# Multipart 请求体组装流程分析

## 概述

Hoppscotch 支持两种主要的请求发送路径，它们在 multipart 请求体的组装方式上有显著差异：

1. **浏览器路径（Browser Interceptor + Axios Relay）**：使用浏览器原生 `FormData` API，浏览器自动处理边界生成和报文拼装
2. **桌面应用路径（Native Interceptor + Desktop Relay/libcurl）**：通过 IPC 将结构化数据传递到 Rust 后端，由 libcurl 完成最终拼装

---

## 1. 组件命名与调用关系

### 1.1 拦截器（Interceptor）

**位置**：`packages/hoppscotch-common/src/platform/std/kernel-interceptors/`

| 拦截器类名 | ID | 适用场景 | 底层 Relay |
|-----------|-----|---------|-----------|
| `BrowserKernelInterceptorService` | `browser` | Web 版 | Axios Relay (`id: "axios"`) |
| `NativeKernelInterceptorService` | `native` | Desktop 版 | Desktop Relay (`id: "desktop"`) |
| `ProxyKernelInterceptorService` | `proxy` | 代理模式 | - |
| `ExtensionKernelInterceptorService` | `extension` | 浏览器扩展 | - |
| `AgentKernelInterceptorService` | `agent` | Agent 模式 | - |

> 不存在 `DesktopKernelInterceptorService`，桌面端使用的是 `NativeKernelInterceptorService`。

### 1.2 Relay 实现

**位置**：`packages/hoppscotch-kernel/src/relay/impl/`

| Relay 实现 | ID | 核心技术 | 支持 multipart |
|-----------|-----|---------|---------------|
| Web/Axios Relay | `axios` | Axios + 浏览器 | 是（FormData 透传） |
| Desktop Relay | `desktop` | Tauri + libcurl | 是 |

### 1.3 调用关系

```
浏览器路径：
[BrowserKernelInterceptorService]
    ↓ (id: "browser")
[Relay.execute()]
    ↓
[getModule("relay")]
    ↓
[window.__KERNEL__.relay]
    ↓
[WEB_RELAY_IMPLS.v1.api] (id: "axios")

桌面路径：
[NativeKernelInterceptorService]
    ↓ (id: "native")
[Relay.execute()]
    ↓
[getModule("relay")]
    ↓
[window.__KERNEL__.relay]
    ↓
[DESKTOP_RELAY_IMPLS.v1.api] (id: "desktop")
```

---

## 2. 数据收集与预处理

### 2.1 前端数据模型

**文件**：`packages/hoppscotch-data/src/rest/v/9/body.ts`

```typescript
export const FormDataKeyValue = z
  .object({
    key: z.string(),
    active: z.boolean(),
    contentType: z.string().optional().catch(undefined),
  })
  .and(
    z.union([
      z.object({ isFile: z.literal(true), value: z.array(z.instanceof(Blob).nullable()).catch([]) }),
      z.object({ isFile: z.literal(false), value: z.string() }),
    ])
  )
```

### 2.2 关键预处理步骤

**文件**：`packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts:300-337`

```typescript
if (request.body.contentType === "multipart/form-data") {
  return pipe(
    request.body.body ?? [],
    // 1. 过滤：移除空键、未激活、无效文件
    A.filter(
      (x) =>
        x.key !== "" &&
        x.active &&
        (typeof x.value === "string" ||
          (x.value.length > 0 && x.value[0] instanceof File))
    ),
    // 2. 排序：文本字段在前，文件字段在后
    arraySort((a, b) => {
      if (a.isFile) return 1
      if (b.isFile) return -1
      return 0
    }),
    // 3. 展开：多文件数组拆分为独立条目
    arrayFlatMap((x) =>
      x.isFile
        ? (Array.isArray(x.value) ? x.value : [x.value]).map((v) => ({
            key: parseTemplateString(x.key, envVariables),
            value: v as string | Blob,
            contentType: x.contentType,
          }))
        : [
            {
              key: parseTemplateString(x.key, envVariables),
              value: parseTemplateString(x.value, envVariables),
              contentType: x.contentType,
            },
          ]
    ),
    // 4. 转换为 FormData 对象
    toFormData
  )
}
```

**预处理要点**：
- **过滤**：只保留激活且有效的字段
- **排序**：文本字段优先，文件字段靠后（影响最终报文顺序）
- **展开**：多文件数组拆分为独立 FormData 条目
- **环境变量解析**：字段名和值中的环境变量会被替换

---

## 3. FormData 转换

**文件**：`packages/hoppscotch-common/src/helpers/functional/formData.ts:1-27`

```typescript
type FormDataEntry = {
  key: string
  contentType?: string
  value: string | Blob
}

export const toFormData = (values: FormDataEntry[]) => {
  const formData = new FormData()

  values.forEach(({ key, value, contentType }) => {
    if (contentType) {
      formData.append(
        key,
        new Blob([value], { type: contentType }),
        key
      )
      return
    }
    formData.append(key, value)
  })

  return formData
}
```

---

## 4. 路径一：Browser Interceptor + Axios Relay

### 4.1 数据流

```
用户输入表单
    ↓
[BodyParameters.vue] 收集
    ↓
[getFinalBodyFromRequest] 预处理
    ↓
[toFormData] 转换为 FormData 对象
    ↓
[transformContent] 包装为 ContentType
    ↓
[BrowserKernelInterceptorService] (id: "browser")
    ↓
[Relay.execute()] → Axios Relay (id: "axios")
    ↓
Axios 发送请求（config.data = FormData 对象）
    ↓
浏览器自动拼装 multipart 报文
```

### 4.2 边界值生成时机

**关键组件**：浏览器内置的 HTTP 栈（由 Axios 调用）

| 阶段 | 边界值处理 |
|------|------------|
| **FormData 创建时** | 无边界值，仅在内存中存储结构化数据 |
| **Axios 接收 FormData 时** | 仍无边界值，Axios 透传给浏览器 |
| **浏览器发送请求前** | 浏览器自动生成唯一的 boundary 字符串 |
| **报文拼装时** | 浏览器将 boundary 写入 Content-Type 头和请求体 |

### 4.3 浏览器自动拼装的报文结构

```
[HTTP Header]
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

[Request Body]
------WebKitFormBoundary7MA4YWxkTrZu0gW\r\n
Content-Disposition: form-data; name="username"\r\n
\r\n
john_doe\r\n
------WebKitFormBoundary7MA4YWxkTrZu0gW\r\n
Content-Disposition: form-data; name="avatar"; filename="a.png"\r\n
Content-Type: image/png\r\n
\r\n
[二进制图片数据]\r\n
------WebKitFormBoundary7MA4YWxkTrZu0gW--\r\n
```

---

## 5. 路径二：Native Interceptor + Desktop Relay/libcurl

### 5.1 完整数据流

```
用户输入表单
    ↓
[BodyParameters.vue] 收集
    ↓
[getFinalBodyFromRequest] → FormData 对象
    ↓
[transformContent] → { kind: "multipart", content: FormData }
    ↓
[NativeKernelInterceptorService] (id: "native")
    ↓
[relayRequestToNativeAdapter] 序列化
    ↓
[makeFormDataSerializable] 转换为可传输格式
    ↓
IPC 跨进程传输到 Rust 后端
    ↓
[ContentHandler.set_multipart_content]
    ↓
[curl::easy::Form] 构建表单
    ↓
libcurl 发送请求时自动拼装
```

### 5.2 Native 拦截器执行流程

**文件**：`packages/hoppscotch-common/src/platform/std/kernel-interceptors/native/index.ts:179-226`

```typescript
private async executeRequest(
  request: RelayRequest,
  setRelayExecution: (execution: { cancel: () => Promise<void> }) => void
): Promise<E.Either<any, RelayResponse>> {
  // 1. 预处理请求
  const effectiveRequest = this.store.completeRequest(
    preProcessRelayRequest(request)
  )

  // 2. 转换为 native 格式（包含 FormData 序列化）
  const nativeRequest = await relayRequestToNativeAdapter(
    effectiveRequestWithUserAgent
  )

  // 3. 后处理（superjson 序列化）
  const postProcessedRequest = postProcessRelayRequest(nativeRequest)

  // 4. 调用 Desktop Relay (id: "desktop")
  const relayExecution = Relay.execute(postProcessedRequest)

  return await relayExecution.response
}
```

### 5.3 FormData 序列化（跨进程传输）

**文件**：`packages/hoppscotch-kernel/src/relay/v/1.ts:724-761`

```typescript
const makeFormDataSerializable = async (
  formData: FormData
): Promise<[string, FormDataValue[]][]> => {
  const m = new Map<string, FormDataValue[]>()

  for (const [key, value] of formData.entries()) {
    if (value instanceof File || value instanceof Blob) {
      const buffer = await value.arrayBuffer()
      const fileEntry: FormDataValue = {
        kind: "file",
        filename: value instanceof File ? value.name : "unknown",
        contentType: value.type || "application/octet-stream",
        data: new Uint8Array(buffer),
      }
      m.has(key) ? m.get(key)!.push(fileEntry) : m.set(key, [fileEntry])
    } else {
      const textEntry: FormDataValue = { kind: "text", value: value.toString() }
      m.has(key) ? m.get(key)!.push(textEntry) : m.set(key, [textEntry])
    }
  }

  return Array.from(m.entries())
}
```

---

## 6. 同名字段顺序变化：完整分步示例

### 6.1 实际交错输入

用户在 UI 中依次添加以下字段（同名字段 `file` 交错文本与文件）：

```
输入顺序（UI 中的行序）：
  #1  key="file", isFile=false, value="desc"        (文本)
  #2  key="file", isFile=true,  value=[file1.png]    (文件)
  #3  key="name", isFile=false, value="john"          (文本)
  #4  key="file", isFile=true,  value=[file2.txt]     (文件)
  #5  key="tag",  isFile=false, value="profile"       (文本)
```

### 6.2 步骤一：EffectiveURL 预处理排序

**代码**（`EffectiveURL.ts:312-316`）：

```typescript
arraySort((a, b) => {
  if (a.isFile) return 1
  if (b.isFile) return -1
  return 0
})
```

排序是**稳定排序**（`Array.prototype.sort` 在 ES2019+ 规范中保证稳定），因此：
- 文本字段之间保持原序
- 文件字段之间保持原序
- 所有文本字段移到所有文件字段之前

```
排序后：
  #1  key="file", isFile=false, value="desc"        (文本，原 #1)
  #3  key="name", isFile=false, value="john"          (文本，原 #3)
  #5  key="tag",  isFile=false, value="profile"       (文本，原 #5)
  #2  key="file", isFile=true,  value=[file1.png]     (文件，原 #2)
  #4  key="file", isFile=true,  value=[file2.txt]     (文件，原 #4)
```

> **关键观察**：排序后，同名的 `file` 文本字段（原 #1）和 `file` 文件字段（原 #2、#4）被分开了，文本的 `file` 排在最前面，文件的 `file` 排在最后面。

### 6.3 步骤二：展开 + toFormData（FormData.append 顺序）

`arrayFlatMap` 将多文件数组拆分后，按排序结果依次调用 `formData.append()`：

```
formData.append("file", "desc")          ← 文本
formData.append("name", "john")          ← 文本
formData.append("tag", "profile")        ← 文本
formData.append("file", file1.png)       ← 文件
formData.append("file", file2.txt)       ← 文件
```

### 6.4 步骤三：FormData.entries() 迭代

`FormData.entries()` 按 `append()` 顺序返回所有条目，**包括同名条目**：

```
1. ["file", "desc"]
2. ["name", "john"]
3. ["tag", "profile"]
4. ["file", file1.png]
5. ["file", file2.txt]
```

### 6.5 步骤四：Map 聚合（`makeFormDataSerializable`）

Map 以 key 为分组依据，**首次出现的 key 决定其在 Map 中的位置**，后续同 key 追加到该 key 的值数组末尾：

```typescript
// 遍历过程
第1次：m.set("file", [textEntry("desc")])           // "file" 首次出现，位置确定
第2次：m.set("name", [textEntry("john")])            // "name" 首次出现
第3次：m.set("tag", [textEntry("profile")])           // "tag" 首次出现
第4次：m.get("file")!.push(fileEntry(file1.png))      // "file" 已存在，追加
第5次：m.get("file")!.push(fileEntry(file2.txt))      // "file" 已存在，追加

// Map 内部状态
Map {
  "file" → [textEntry("desc"), fileEntry(file1.png), fileEntry(file2.txt)],
  "name" → [textEntry("john")],
  "tag"  → [textEntry("profile")]
}
```

### 6.6 步骤五：Array.from(m.entries()) 输出

```typescript
[
  ["file", [textEntry("desc"), fileEntry(file1.png), fileEntry(file2.txt)]],
  ["name", [textEntry("john")]],
  ["tag",  [textEntry("profile")]]
]
```

### 6.7 步骤六：Rust 侧遍历拼装

**代码**（`content.rs:213-263`）：

```rust
for (key, values) in content {
    for value in values {
        match value {
            FormValue::Text { value } => { form.part(key).contents(value.as_bytes()).add()?; }
            FormValue::File { filename, content_type, data } => {
                form.part(key).buffer(&filename, data.to_vec())
                    .content_type(&content_type.to_string()).add()?;
            }
        }
    }
}
```

**最终报文中的字段顺序**：

```
--{boundary}\r\n
Content-Disposition: form-data; name="file"\r\n        ← 文本 "desc"
\r\ndesc\r\n
--{boundary}\r\n
Content-Disposition: form-data; name="file"; filename="file1.png"\r\n
Content-Type: image/png\r\n
\r\n[二进制]\r\n
--{boundary}\r\n
Content-Disposition: form-data; name="file"; filename="file2.txt"\r\n
Content-Type: text/plain\r\n
\r\n[二进制]\r\n
--{boundary}\r\n
Content-Disposition: form-data; name="name"\r\n
\r\njohn\r\n
--{boundary}\r\n
Content-Disposition: form-data; name="tag"\r\n
\r\nprofile\r\n
--{boundary}--\r\n
```

### 6.8 顺序变化汇总

| 阶段 | 字段顺序 | 同名 `file` 字段分布 |
|------|---------|---------------------|
| **UI 原始输入** | `file(text) → file(file) → name → file(file) → tag` | 交错分布 |
| **EffectiveURL 排序后** | `file(text) → name → tag → file(file) → file(file)` | 文本 `file` 在前，文件 `file` 在后 |
| **FormData entries** | `file(text) → name → tag → file(file) → file(file)` | 同排序后（FormData 保持 append 顺序） |
| **Map 聚合后** | `file([text, file, file]) → name → tag` | 同名 `file` 被合并为一个键 |
| **最终报文** | `file(text) → file(file) → file(file) → name → tag` | 同名 `file` 连续排列 |

### 6.9 两条关于预处理排序影响的结论

**结论一：两条发送路径均受预处理排序影响，且排序结果相同**

EffectiveURL 的 `arraySort` 排序发生在 `toFormData` 之前，无论请求走浏览器路径还是 Desktop 路径，FormData 中的字段顺序都已经被排序决定。两条路径接收到的是同一个排序后的 FormData 对象，因此**预处理排序对两条路径的影响是一致的**——文本字段始终排列在文件字段之前。

**结论二：预处理排序是同名字段被连续排列的根因之一，但两条路径的"连续"含义不同**

- 在**浏览器路径**中，FormData 保持排序后的 append 顺序，同名但不同类型（文本/文件）的字段因排序而已被分开，但 `FormData.entries()` 仍可在不同键之间保持交错。同键字段如果排序后不相邻（如 `file(text)` 和 `file(file)` 之间隔着 `name`、`tag`），在浏览器路径中它们**仍然被其他键隔开**。
- 在**Desktop 路径**中，Map 聚合会将同键字段**强制合并**为连续条目，无论它们在 FormData 中是否被其他键隔开。因此 Desktop 路径中同名 `file` 的三个值（text、file、file）必然连续排列，而浏览器路径中 `file(text)` 和 `file(file)` 之间隔着 `name` 和 `tag`。

---

## 7. Content-Type 与 boundary：代码事实、库行为推断与需实测项

### 7.1 代码事实（可直接从源码验证）

**事实 1：`getComputedBodyHeaders` 预置不含 boundary 的 Content-Type**

**文件**：`EffectiveURL.ts:130-137`

```typescript
return [
  {
    active: true,
    key: "content-type",
    value: req.body.contentType,  // 值为 "multipart/form-data"，不含 boundary
    description: "",
  },
]
```

预置的 Content-Type 值就是 `req.body.contentType`，即 `"multipart/form-data"`，不包含 `; boundary=...`。

**事实 2：用户手动设置 Content-Type 时，跳过自动生成**

**文件**：`EffectiveURL.ts:92-97`

```typescript
if (
  req.headers.find(
    (req) => req.active && req.key.toLowerCase() === "content-type"
  )
)
  return []
```

**事实 3：Rust 侧 `set_form_content` 不写入 Content-Type 头**

**文件**：`content.rs:206-209`

```rust
/* TODO: Look into reintroducing this when auth handling is done by kernel */
// let mut headers = HashMap::new();
// headers.insert("content-type".to_string(), media_type.to_string());
// self.merge_headers(headers);
```

Content-Type 头的写入被注释掉了，`merge_headers` 不会被调用。因此 Rust 侧不会主动设置 `content-type` 头。

**事实 4：Axios Relay 将 `request.content?.content` 直接赋给 `config.data`**

**文件**：`packages/hoppscotch-kernel/src/relay/impl/web/v\1.ts:130`

```typescript
data: request.content?.content,
```

当 `content.kind === "multipart"` 时，`content.content` 是一个 `FormData` 对象。Axios 接收到 `FormData` 后透传给浏览器。

**事实 5：预置的 Content-Type 头会进入 Axios config.headers**

`request.headers` 包含了 `getComputedBodyHeaders` 生成的 `content-type: multipart/form-data`（不含 boundary），Axios 将其传给浏览器的 XHR/fetch。

### 7.2 库行为推断（基于库文档和通用知识，非本仓库代码直接证明）

**推断 1：浏览器自动追加 boundary**

当 XHR/fetch 的 body 是 FormData 对象时，浏览器会：
- 忽略或覆盖预置的 `Content-Type: multipart/form-data`
- 自动生成完整的 `Content-Type: multipart/form-data; boundary=...`
- 使用该 boundary 拼装请求体

这是浏览器标准行为，但**本仓库代码未显式处理此逻辑**。

**推断 2：Axios 对 FormData 的 Content-Type 处理**

Axios 检测到 `data` 是 `FormData` 实例时，会删除用户设置的 `Content-Type` 头，让浏览器自动生成。这是 Axios 的内置行为。

**推断 3：libcurl 的 `httppost` 自动生成 boundary 和 Content-Type**

当调用 `handle.httppost(form)` 后，libcurl 在 `curl_easy_perform()` 时会：
- 自动生成 boundary 字符串
- 自动设置 `Content-Type: multipart/form-data; boundary=...` 请求头
- 使用该 boundary 拼装请求体

由于 Rust 侧的 `merge_headers` 被注释掉，没有手动的 `content-type` 头与 libcurl 自动生成的头冲突。

**推断 4：如果用户预置的 `Content-Type` 含 boundary，底层库的行为不确定**

当 `request.headers` 中包含用户手动设置的 `Content-Type: multipart/form-data; boundary=custom123` 时：
- 浏览器路径：Axios 可能删除该头让浏览器重新生成，也可能保留它
- Desktop 路径：该头会通过 `HeadersBuilder.add_headers` 写入 curl 的 `http_headers` 列表，可能与 libcurl 自动生成的头**同时存在**，造成重复或冲突

### 7.3 需实测项（无法仅从代码推断）

| 测试项 | 测试方法 | 预期结果 |
|--------|---------|---------|
| **浏览器路径下预置的 `Content-Type` 是否被浏览器覆盖** | 使用 Browser 拦截器发送 multipart 请求，在 DevTools 中观察实际发出的 Content-Type 头 | 浏览器生成的完整头应包含 boundary |
| **Desktop 路径下预置的 `Content-Type` 是否与 libcurl 冲突** | 使用 Native 拦截器发送 multipart 请求，通过 Wireshark 或 curl verbose 观察实际发出的头部 | libcurl 应自动生成含 boundary 的头 |
| **用户手动设置 `Content-Type` 含 boundary 时的行为** | 在请求头中手动添加 `Content-Type: multipart/form-data; boundary=custom123`，分别用两条路径发送 | 需实测确认是否产生重复头或解析错误 |
| **Axios 是否删除预置的 `Content-Type`** | 在 Axios Relay 的 `config` 中设置断点，观察 `headers` 中 `content-type` 在发送前的值 | Axios 文档声称会删除，需实测确认 |
| **libcurl `httppost` 与手动 `http_headers` 中 `content-type` 的优先级** | 在 Rust 侧同时设置 `httppost(form)` 和 `http_headers(list)`（含 content-type），观察实际发出哪个 | 需实测，libcurl 文档未明确说明优先级 |

---

## 8. 文本字段 vs 文件字段：拼装细节对比

### 8.1 文本字段拼装

| 组件 | 内容 | 说明 |
|------|------|------|
| **边界行** | `--{boundary}\r\n` | 每个字段开始 |
| **Content-Disposition** | `form-data; name="字段名"` | 必填 |
| **Content-Type** | 无 | 文本字段通常不需要 |
| **空行** | `\r\n` | 分隔头和内容 |
| **内容** | 文本字符串 | UTF-8 编码 |
| **结尾** | `\r\n` | 字段结束 |

### 8.2 文件字段拼装

| 组件 | 内容 | 说明 |
|------|------|------|
| **边界行** | `--{boundary}\r\n` | 与文本字段相同 |
| **Content-Disposition** | `form-data; name="字段名"; filename="文件名"` | 额外的 filename 参数 |
| **Content-Type** | `MIME 类型` | 如 `image/png`、`application/pdf` 等 |
| **空行** | `\r\n` | 分隔头和内容 |
| **内容** | 二进制数据 | 原始字节，不编码 |
| **结尾** | `\r\n` | 字段结束 |

---

## 9. 两条路径对比总结

| 对比项 | Browser Interceptor + Axios Relay | Native Interceptor + Desktop Relay |
|--------|----------------------------------|-----------------------------------|
| **拦截器类名** | `BrowserKernelInterceptorService` | `NativeKernelInterceptorService` |
| **拦截器 ID** | `browser` | `native` |
| **Relay 实现 ID** | `axios` | `desktop` |
| **边界生成者** | 浏览器内置 HTTP 栈 | libcurl 库内部 |
| **生成时机** | 发送请求瞬间 | `curl_easy_perform()` 执行时 |
| **拼装位置** | JavaScript 运行时 + 浏览器内核 | Rust 后端 + libcurl |
| **数据传输** | 内存中 FormData 对象 | IPC 序列化的结构化数据 |
| **Content-Type 头** | 浏览器自动设置 | libcurl 自动设置 |
| **字段顺序** | FormData append 顺序 | Map 插入顺序（预处理时排序） |
| **二进制处理** | 浏览器直接处理 Blob | 转换为 Uint8Array 跨进程传输 |
| **适用场景** | Web 版 Hoppscotch | 桌面版 Hoppscotch |
| **同名字段处理** | 保持排序后的交错顺序 | Map 合并为连续条目 |
| **Rust 侧 Content-Type 写入** | 不适用 | 被注释掉，由 libcurl 自动生成 |

---

## 10. 边界值（Boundary）深入解析

### 10.1 边界值格式规范

根据 RFC 2046 Section 5.1.1：

```
boundary := 0*69<bchars> bcharsnospace
bchars := bcharsnospace / " "
bcharsnospace := DIGIT / ALPHA / "'" / "(" / ")" / "+" / "_" / "," / "-" / "." / "/" / ":" / "=" / "?"
```

### 10.2 典型边界值示例

| 生成源 | 示例 |
|--------|------|
| **Chrome/WebKit** | `----WebKitFormBoundary7MA4YWxkTrZu0gW` |
| **Firefox** | `---------------------------19813753121051141521880229351` |
| **libcurl** | `------------------------d74496d66958873e` |

### 10.3 边界值唯一性保证

- 浏览器和 libcurl 都使用随机数生成器确保边界值不会出现在内容中
- 理论上存在冲突概率，但实际应用中可忽略

---

## 11. CRLF 与换行处理

### 11.1 规范要求（RFC 7578 Section 4.1）

> The parts are separated by the boundary delimiter line. Each part is preceded by a boundary delimiter line, and the last part is followed by a closing boundary delimiter line. Each boundary delimiter line must be followed immediately by a CRLF.

### 11.2 拼装时的 CRLF 插入点

```
--{boundary}\r\n           ← 边界后必须有 CRLF
头字段\r\n                 ← 每个头字段后有 CRLF
\r\n                       ← 头结束后有空行（额外的 CRLF）
内容数据                   ← 内容本身
\r\n                       ← 内容后有 CRLF
--{boundary}--\r\n         ← 结束边界后有 CRLF
```

### 11.3 常见坑点

1. **缺失结尾 CRLF**：某些服务器对格式要求严格，缺少会导致解析失败
2. **LF 代替 CRLF**：Unix 风格换行在 multipart 中是错误的
3. **多余空行**：边界之间的多余空行可能被当作内容的一部分

---

## 12. 相关代码文件索引

| 文件路径 | 职责 |
|----------|------|
| `packages/hoppscotch-data/src/rest/v/9/body.ts` | 表单数据模型定义 |
| `packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts` | 请求体预处理、排序、环境变量解析、Content-Type 预置 |
| `packages/hoppscotch-common/src/helpers/functional/formData.ts` | FormData 对象构建 |
| `packages/hoppscotch-common/src/helpers/kernel/common/content.ts` | `transformContent`：将 effectiveFinalBody 转为 ContentType |
| `packages/hoppscotch-kernel/src/relay/v/1.ts` | FormData 序列化、`content.multipart()` 构造器 |
| `packages/hoppscotch-kernel/src/relay/impl/web/v\1.ts` | Axios Relay 实现（id: "axios"） |
| `packages/hoppscotch-kernel/src/relay/impl/desktop/v\1.ts` | Desktop Relay 实现（id: "desktop"） |
| `packages/hoppscotch-common/src/platform/std/kernel-interceptors/browser/index.ts` | Browser 拦截器（id: "browser"） |
| `packages/hoppscotch-common/src/platform/std/kernel-interceptors/native/index.ts` | Native 拦截器（id: "native"） |
| `packages/hoppscotch-desktop/plugin-workspace/relay/src/content.rs` | Rust 侧内容处理、libcurl Form 构建 |
| `packages/hoppscotch-desktop/plugin-workspace/relay/src/header.rs` | Rust 侧 HeadersBuilder，将 HashMap 写入 curl List |
| `packages/hoppscotch-desktop/plugin-workspace/relay/src/request.rs` | Rust 侧请求准备，协调 content/header/auth |
| `packages/hoppscotch-desktop/plugin-workspace/relay/src/interop.rs` | Rust 侧数据结构定义 |
| `packages/hoppscotch-kernel/src/index.ts` | Kernel 初始化，Relay 实现选择 |

---

## 13. 参考规范

- **RFC 7578**：Returning Values from Forms: multipart/form-data
- **RFC 2046**：Multipurpose Internet Mail Extensions (MIME) Part Two: Media Types
- **RFC 2388**：Returning Values from Forms: multipart/form-data（已被 RFC 7578 取代）
- **ECMAScript 2019**：`Array.prototype.sort` 稳定性保证
