# Multipart 请求体组装流程分析

## 概述

Hoppscotch 支持两种主要的请求发送路径，它们在 multipart 请求体的组装方式上有显著差异：

1. **浏览器路径（Browser Interceptor + Axios Relay）**：使用浏览器原生 `FormData` API，浏览器自动处理边界生成和报文拼装
2. **桌面应用路径（Native Interceptor + Desktop Relay/libcurl）**：通过 IPC 将结构化数据传递到 Rust 后端，由 libcurl 完成最终拼装

---

## 1. 组件命名与调用关系核实

### 1.1 拦截器（Interceptor）

**位置**：`packages/hoppscotch-common/src/platform/std/kernel-interceptors/`

| 拦截器类名 | ID | 适用场景 | 底层 Relay |
|-----------|-----|---------|-----------|
| `BrowserKernelInterceptorService` | `browser` | Web 版 | Axios Relay (`id: "axios"`) |
| `NativeKernelInterceptorService` | `native` | Desktop 版 | Desktop Relay (`id: "desktop"`) |
| `ProxyKernelInterceptorService` | `proxy` | 代理模式 | - |
| `ExtensionKernelInterceptorService` | `extension` | 浏览器扩展 | - |
| `AgentKernelInterceptorService` | `agent` | Agent 模式 | - |

> **重要纠正**：不存在 `DesktopKernelInterceptorService`，桌面端使用的是 `NativeKernelInterceptorService`。

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

## 3. FormData 转换（浏览器端）

### 3.1 转换函数

**文件**：`packages/hoppscotch-common/src/helpers/functional/formData.ts:7-27`

```typescript
export const toFormData = (values: FormDataEntry[]) => {
  const formData = new FormData()

  values.forEach(({ key, value, contentType }) => {
    if (contentType) {
      // 有指定 contentType：包装为 Blob，指定文件名
      formData.append(
        key,
        new Blob([value], { type: contentType }),
        key  // 第三个参数作为文件名
      )
      return
    }
    // 普通字段：直接添加
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
Axios 发送请求（data: FormData 对象）
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

**拼装顺序规则**：
1. 每个部分以 `--{boundary}\r\n` 开始
2. 然后是 `Content-Disposition` 头
3. 文件字段额外有 `Content-Type` 头
4. 空行 `\r\n` 分隔头和内容
5. 内容数据
6. `\r\n` 结束该部分
7. 所有部分完成后，以 `--{boundary}--\r\n` 结束

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

**Kernel 初始化**（`packages/hoppscotch-kernel/src/index.ts:46-61`）：

```typescript
export function initKernel(mode?: KernelMode): KernelAPI {
  if (mode === "desktop") {
    const kernel: KernelAPI = {
      // ...
      relay: DESKTOP_RELAY_IMPLS.v1.api,  // 使用 Desktop Relay 实现
      // ...
    }
  }
}
```

### 5.3 FormData 序列化（跨进程传输）

**文件**：`packages/hoppscotch-kernel/src/relay/v/1.ts:724-761`

```typescript
const makeFormDataSerializable = async (
  formData: FormData
): Promise<[string, FormDataValue[]][]> => {
  const m = new Map<string, FormDataValue[]>()  // 使用 Map 保持顺序

  for (const [key, value] of formData.entries()) {
    if (value instanceof File || value instanceof Blob) {
      // 文件：读取二进制数据，提取元数据
      const buffer = await value.arrayBuffer()
      const fileEntry: FormDataValue = {
        kind: "file",
        filename: value instanceof File ? value.name : "unknown",
        contentType: value.type || "application/octet-stream",
        data: new Uint8Array(buffer),
      }
      m.has(key) ? m.get(key)!.push(fileEntry) : m.set(key, [fileEntry])
    } else {
      // 文本：转换为字符串
      const textEntry: FormDataValue = { kind: "text", value: value.toString() }
      m.has(key) ? m.get(key)!.push(textEntry) : m.set(key, [textEntry])
    }
  }

  return Array.from(m.entries())
}
```

### 5.4 同名字段的顺序变化（完整示例）

**关键问题**：`formData.entries()` 按 `append()` 顺序返回所有条目，但 `Map.set()` 会将同名字段**按键分组**，导致**同名字段被合并**，改变相对顺序。

#### 示例：同名字段交错场景

假设原始 `FormData.append()` 顺序如下（文本-文件-文本交错）：

```typescript
// 原始 append 顺序（模拟用户在 UI 中的输入顺序）
formData.append("file", file1)       // 第1个 file 字段（文件）
formData.append("file", "text1")     // 第2个 file 字段（文本）
formData.append("file", file2)       // 第3个 file 字段（文件）
formData.append("name", "john")      // name 字段
```

**阶段一：FormData.entries() 迭代（原始顺序）**

```
formData.entries() 输出顺序：
1. ["file", file1]
2. ["file", "text1"]
3. ["file", file2]
4. ["name", "john"]
```

**阶段二：Map 聚合（按键分组）**

```typescript
// 遍历过程
const m = new Map()
m.set("file", [file1])              // 第1次：新增键 "file"
m.get("file")!.push(textEntry1)     // 第2次：同键追加
m.get("file")!.push(file2)          // 第3次：同键追加
m.set("name", [textEntry])          // 第4次：新增键 "name"

// Map 内部状态
Map {
  "file" → [file1, textEntry1, file2],  // 同名字段被合并为数组
  "name" → [textEntry]
}
```

**阶段三：Array.from(m.entries()) 输出**

```typescript
// 最终序列化结果
[
  ["file", [file1, textEntry1, file2]],  // 所有 "file" 字段值合并在一起
  ["name", [textEntry]]
]
```

**阶段四：Rust 侧遍历拼装（`content.rs:267-285`）**

```rust
for (key, values) in content {
    for value in values {
        match value {
            FormValue::Text { value } => { /* 添加文本字段 */ }
            FormValue::File { .. } => { /* 添加文件字段 */ }
        }
    }
}
```

**最终报文顺序（Rust 侧）**：

```
--{boundary}\r\n
Content-Disposition: form-data; name="file"; filename="file1.txt"\r\n
...
--{boundary}\r\n
Content-Disposition: form-data; name="file"\r\n
...
--{boundary}\r\n
Content-Disposition: form-data; name="file"; filename="file2.txt"\r\n
...
--{boundary}\r\n
Content-Disposition: form-data; name="name"\r\n
...
```

**顺序变化对比表**：

| 阶段 | 顺序 | 说明 |
|------|------|------|
| **原始 append 顺序** | `file(file) → file(text) → file(file) → name` | 同名字段交错分布 |
| **Map 转换后** | `file([file, text, file]) → name` | 同名字段被合并到一起 |
| **最终报文顺序** | `file(file) → file(text) → file(file) → name` | 同名字段连续排列 |

> **重要结论**：同名字段的内部值顺序保持不变，但**同名字段会被连续排列**，而不是保持原始的交错分布。这符合 RFC 7578 规范的要求，但与原始 UI 输入顺序可能不同。

**序列化后的数据结构**：

```typescript
// TypeScript 侧
[
  ["username", [{ kind: "text", value: "john_doe" }]],
  ["avatar", [{ kind: "file", filename: "a.png", contentType: "image/png", data: Uint8Array }]]
]

// Rust 侧（interop.rs）
pub enum FormValue {
    Text { value: String },
    File { filename: String, content_type: MediaType, data: Bytes },
}
pub type FormData = Vec<(String, Vec<FormValue>)>;
```

### 5.5 Rust 侧 - libcurl 表单构建

**文件**：`packages/hoppscotch-desktop/plugin-workspace/relay/src/content.rs:201-347`

```rust
fn set_form_content(
    &mut self,
    content: &Vec<(String, Vec<FormValue>)>,
    media_type: &MediaType,
) -> Result<()> {
    // 创建 curl Form 构建器
    let mut form = curl::easy::Form::new();

    for (key, values) in content {
        for value in values {
            match value {
                // 文本字段
                FormValue::Text { value: text } => {
                    form.part(key)
                        .contents(text.as_bytes())  // 文本内容字节
                        .add()?;
                }
                // 文件字段
                FormValue::File { filename, content_type, data } => {
                    form.part(key)
                        .buffer(&filename, data.to_vec())   // 文件名 + 二进制数据
                        .content_type(&content_type.to_string())  // MIME 类型
                        .add()?;
                }
            }
        }
    }

    // 绑定到 curl handle
    self.handle.httppost(form)?;
    Ok(())
}
```

### 5.6 libcurl 边界值生成时机

**关键组件**：libcurl 库的内部实现

| 阶段 | 边界值处理 |
|------|------------|
| **Form::new() 创建时** | 无边界值，仅构建内存结构 |
| **form.part().add() 时** | 仍无边界值，添加字段到内部列表 |
| **handle.httppost(form) 时** | 绑定到请求，不生成边界 |
| **curl_easy_perform() 执行时** | libcurl 内部生成 boundary，拼装完整报文 |

### 5.7 libcurl 拼装的报文结构

libcurl 生成的报文格式与浏览器一致，符合 RFC 7578 规范：

```
[HTTP Header]
Content-Type: multipart/form-data; boundary=------------------------d74496d66958873e

[Request Body]
--------------------------d74496d66958873e\r\n
Content-Disposition: form-data; name="username"\r\n
\r\n
john_doe\r\n
--------------------------d74496d66958873e\r\n
Content-Disposition: form-data; name="avatar"; filename="a.png"\r\n
Content-Type: image/png\r\n
\r\n
[二进制图片数据]\r\n
--------------------------d74496d66958873e--\r\n
```

---

## 6. Content-Type 头预置与 boundary 生成的关系

### 6.1 Content-Type 头预置逻辑

**文件**：`packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts:83-138`

```typescript
export const getComputedBodyHeaders = (
  req: HoppRESTRequest | { auth: HoppRESTAuth; headers: HoppRESTHeaders }
): HoppRESTHeader[] => {
  // 如果用户已手动设置 Content-Type，则跳过自动生成
  if (
    req.headers.find(
      (req) => req.active && req.key.toLowerCase() === "content-type"
    )
  )
    return []

  // ... 其他类型处理 ...

  // 自动生成 Content-Type 头（不带 boundary）
  return [
    {
      active: true,
      key: "content-type",
      value: req.body.contentType,  // "multipart/form-data"
      description: "",
    },
  ]
}
```

### 6.2 预置 Content-Type 与 boundary 生成的交互

| 场景 | 预置头内容 | 底层库行为 | 最终 Content-Type 头 |
|------|-----------|-----------|---------------------|
| **用户未手动设置** | `multipart/form-data` | 浏览器/libcurl 自动添加 boundary | `multipart/form-data; boundary=xxxx` |
| **用户手动设置（无 boundary）** | 用户自定义值 | 浏览器/libcurl 添加 boundary | `{用户值}; boundary=xxxx` |
| **用户手动设置（含 boundary）** | 用户自定义值（含 boundary） | 浏览器/libcurl 使用用户提供的 boundary | `{用户值}` |

### 6.3 浏览器（Axios）处理逻辑

1. **预置头**：`Content-Type: multipart/form-data`
2. **发送时**：浏览器检测到 body 是 FormData，自动在 header 后追加 `; boundary=...`
3. **最终 header**：`Content-Type: multipart/form-data; boundary=----WebKitFormBoundary...`

### 6.4 libcurl 处理逻辑

1. **预置头**：`Content-Type: multipart/form-data`
2. **执行时**：libcurl 的 `curl_easy_perform()` 内部生成 boundary
3. **最终 header**：`Content-Type: multipart/form-data; boundary=------------------------...`

### 6.5 可验证的结论

**结论 1：预置的 Content-Type 不包含 boundary**
- boundary 是底层库在发送时**动态生成**的
- JavaScript 层无法获取或控制实际的 boundary 值

**结论 2：用户手动设置优先**
- 如果用户手动设置了 Content-Type 头，系统不会自动覆盖
- 用户可以手动指定 boundary，但不推荐（底层库生成的更安全）

**结论 3：boundary 唯一性由底层库保证**
- 浏览器和 libcurl 都使用随机数生成器确保 boundary 不会出现在内容中
- 理论上存在冲突概率，但实际应用中可忽略

**结论 4：预置头仅起告知作用**
- 预置头的作用是告知服务器请求体类型
- 具体 boundary 由底层库在发送前决定

---

## 7. 文本字段 vs 文件字段：拼装细节对比

### 7.1 文本字段拼装

| 组件 | 内容 | 说明 |
|------|------|------|
| **边界行** | `--{boundary}\r\n` | 每个字段开始 |
| **Content-Disposition** | `form-data; name="字段名"` | 必填 |
| **Content-Type** | 无 | 文本字段通常不需要 |
| **空行** | `\r\n` | 分隔头和内容 |
| **内容** | 文本字符串 | UTF-8 编码 |
| **结尾** | `\r\n` | 字段结束 |

### 7.2 文件字段拼装

| 组件 | 内容 | 说明 |
|------|------|------|
| **边界行** | `--{boundary}\r\n` | 与文本字段相同 |
| **Content-Disposition** | `form-data; name="字段名"; filename="文件名"` | 额外的 filename 参数 |
| **Content-Type** | `MIME 类型` | 如 `image/png`、`application/pdf` 等 |
| **空行** | `\r\n` | 分隔头和内容 |
| **内容** | 二进制数据 | 原始字节，不编码 |
| **结尾** | `\r\n` | 字段结束 |

### 7.3 字段顺序保证

两种路径都通过特定机制保证字段顺序：

1. **浏览器路径**：`FormData.entries()` 按 `append()` 顺序迭代
2. **Desktop 路径**：使用 `Map` 存储（ES2015 保证插入顺序），序列化后传给 Rust

**排序规则**（`EffectiveURL.ts:312-316`）：

```typescript
// 文本字段在前，文件字段在后
arraySort((a, b) => {
  if (a.isFile) return 1    // 文件排后面
  if (b.isFile) return -1   // 非文件排前面
  return 0
})
```

---

## 8. 两条路径对比总结

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
| **同名字段处理** | 保持原始交错顺序 | 合并为连续条目 |

### 8.1 共同点

1. **都不手动拼装报文**：两种路径都依赖底层库（浏览器/libcurl）处理边界生成和报文拼装
2. **字段排序相同**：预处理时都将文本字段排在前面，文件字段在后面
3. **最终报文格式一致**：都符合 RFC 7578 规范，服务器端无法区分来源

### 8.2 关键差异

- **边界值可见性**：两种路径下，JavaScript 层都**无法**获取或控制实际的 boundary 值
- **调试难度**：浏览器路径可通过 DevTools Network 面板查看完整报文；Desktop 路径需要启用 curl 调试日志
- **性能特征**：大文件上传时，Desktop 路径的 IPC 传输可能成为瓶颈
- **同名字段处理**：Desktop 路径会将同名字段合并为连续条目，而浏览器路径保持原始交错顺序

---

## 9. 边界值（Boundary）深入解析

### 9.1 边界值格式规范

根据 RFC 2046 Section 5.1.1：

```
boundary := 0*69<bchars> bcharsnospace
bchars := bcharsnospace / " "
bcharsnospace := DIGIT / ALPHA / "'" / "(" / ")" / "+" / "_" / "," / "-" / "." / "/" / ":" / "=" / "?"
```

### 9.2 典型边界值示例

| 生成源 | 示例 |
|--------|------|
| **Chrome/WebKit** | `----WebKitFormBoundary7MA4YWxkTrZu0gW` |
| **Firefox** | `---------------------------19813753121051141521880229351` |
| **libcurl** | `------------------------d74496d66958873e` |
| **Node.js form-data** | `--------------------------${随机串}` |

### 9.3 边界值唯一性保证

- 浏览器和 libcurl 都使用随机数生成器确保边界值不会出现在内容中
- 理论上存在冲突概率，但实际应用中可忽略
- 如果内容恰好包含边界字符串，会导致解析错误（极罕见）

---

## 10. CRLF 与换行处理

### 10.1 规范要求（RFC 7578 Section 4.1）

> The parts are separated by the boundary delimiter line. Each part is preceded by a boundary delimiter line, and the last part is followed by a closing boundary delimiter line. Each boundary delimiter line must be followed immediately by a CRLF.

### 10.2 拼装时的 CRLF 插入点

```
--{boundary}\r\n           ← 边界后必须有 CRLF
头字段\r\n                 ← 每个头字段后有 CRLF
\r\n                       ← 头结束后有空行（额外的 CRLF）
内容数据                   ← 内容本身
\r\n                       ← 内容后有 CRLF
--{boundary}--\r\n         ← 结束边界后有 CRLF
```

### 10.3 常见坑点

1. **缺失结尾 CRLF**：某些服务器对格式要求严格，缺少会导致解析失败
2. **LF 代替 CRLF**：Unix 风格换行在 multipart 中是错误的
3. **多余空行**：边界之间的多余空行可能被当作内容的一部分

---

## 11. 相关代码文件索引

| 文件路径 | 职责 |
|----------|------|
| `packages/hoppscotch-data/src/rest/v/9/body.ts` | 表单数据模型定义 |
| `packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts` | 请求体预处理、排序、环境变量解析 |
| `packages/hoppscotch-common/src/helpers/functional/formData.ts` | FormData 对象构建 |
| `packages/hoppscotch-kernel/src/relay/v/1.ts` | FormData 序列化、跨平台传输格式 |
| `packages/hoppscotch-kernel/src/relay/impl/web/v/1.ts` | Axios Relay 实现（ID: "axios"） |
| `packages/hoppscotch-kernel/src/relay/impl/desktop/v/1.ts` | Desktop Relay 实现（ID: "desktop"） |
| `packages/hoppscotch-common/src/platform/std/kernel-interceptors/browser/index.ts` | Browser 拦截器（ID: "browser"） |
| `packages/hoppscotch-common/src/platform/std/kernel-interceptors/native/index.ts` | Native 拦截器（ID: "native"） |
| `packages/hoppscotch-desktop/plugin-workspace/relay/src/content.rs` | Rust 侧内容处理、libcurl Form 构建 |
| `packages/hoppscotch-desktop/plugin-workspace/relay/src/interop.rs` | Rust 侧数据结构定义 |
| `packages/hoppscotch-kernel/src/index.ts` | Kernel 初始化，Relay 实现选择 |

---

## 12. 参考规范

- **RFC 7578**：Returning Values from Forms: multipart/form-data
- **RFC 2046**：Multipurpose Internet Mail Extensions (MIME) Part Two: Media Types
- **RFC 2388**：Returning Values from Forms: multipart/form-data（已被 RFC 7578 取代）
- **HTML 5.2 Section 4.10.21.8**：Multipart form data
