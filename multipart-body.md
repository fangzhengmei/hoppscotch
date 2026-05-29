# Multipart 请求体组装流程分析

## 概述

Hoppscotch 支持两种主要的请求发送路径，它们在 multipart 请求体的组装方式上有显著差异：

1. **浏览器路径（Browser/Axios）**：使用浏览器原生 `FormData` API，浏览器自动处理边界生成和报文拼装
2. **桌面应用路径（Desktop/Relay/libcurl）**：通过 IPC 将结构化数据传递到 Rust 后端，由 libcurl 完成最终拼装

---

## 1. 数据收集与预处理

### 1.1 前端数据模型

**文件**: `packages/hoppscotch-data/src/rest/v/9/body.ts`

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

### 1.2 关键预处理步骤

**文件**: `packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts:300-337`

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

**预处理要点**:
- **过滤**：只保留激活且有效的字段
- **排序**：文本字段优先，文件字段靠后（影响最终报文顺序）
- **展开**：多文件数组拆分为独立 FormData 条目
- **环境变量解析**：字段名和值中的环境变量会被替换

---

## 2. FormData 转换（浏览器端）

### 2.1 转换函数

**文件**: `packages/hoppscotch-common/src/helpers/functional/formData.ts:7-27`

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

## 3. 路径一：浏览器 Fetch/Axios 路径

### 3.1 数据流

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
[BrowserKernelInterceptorService] 调用 Relay
    ↓
Axios 发送请求（data: FormData 对象）
    ↓
浏览器自动拼装 multipart 报文
```

### 3.2 边界值生成时机

**关键组件**：浏览器内置的 HTTP 栈（由 Axios 调用）

| 阶段 | 边界值处理 |
|------|------------|
| **FormData 创建时** | 无边界值，仅在内存中存储结构化数据 |
| **Axios 接收 FormData 时** | 仍无边界值，Axios 透传给浏览器 |
| **浏览器发送请求前** | 浏览器自动生成唯一的 boundary 字符串 |
| **报文拼装时** | 浏览器将 boundary 写入 Content-Type 头和请求体 |

### 3.3 浏览器自动拼装的报文结构

```
[HTTP Header]
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

[Request Body]
------WebKitFormBoundary7MA4YWxkTrZu0gW\r\n                    ← 边界前缀（-- + boundary）
Content-Disposition: form-data; name="username"\r\n             ← 字段头
\r\n                                                             ← 空行（CRLF）
john_doe\r\n                                                     ← 内容 + CRLF
------WebKitFormBoundary7MA4YWxkTrZu0gW\r\n                    ← 下一个边界
Content-Disposition: form-data; name="avatar"; filename="a.png"\r\n
Content-Type: image/png\r\n
\r\n
[二进制图片数据]\r\n
------WebKitFormBoundary7MA4YWxkTrZu0gW--\r\n                  ← 结束边界（-- + boundary + --）
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

## 4. 路径二：Desktop Relay / libcurl 路径

### 4.1 完整数据流

```
用户输入表单
    ↓
[BodyParameters.vue] 收集
    ↓
[getFinalBodyFromRequest] → FormData 对象
    ↓
[transformContent] → { kind: "multipart", content: FormData }
    ↓
[DesktopKernelInterceptorService] 调用
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

### 4.2 FormData 序列化（跨进程传输）

**文件**: `packages/hoppscotch-kernel/src/relay/v/1.ts:724-761`

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

### 4.3 Rust 侧 - libcurl 表单构建

**文件**: `packages/hoppscotch-desktop/plugin-workspace/relay/src/content.rs:201-347`

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

### 4.4 libcurl 边界值生成时机

**关键组件**：libcurl 库的内部实现

| 阶段 | 边界值处理 |
|------|------------|
| **Form::new() 创建时** | 无边界值，仅构建内存结构 |
| **form.part().add() 时** | 仍无边界值，添加字段到内部列表 |
| **handle.httppost(form) 时** | 绑定到请求，不生成边界 |
| **curl_easy_perform() 执行时** | libcurl 内部生成 boundary，拼装完整报文 |

### 4.5 libcurl 拼装的报文结构

libcurl 生成的报文格式与浏览器一致，符合 RFC 7578 规范：

```
[HTTP Header]
Content-Type: multipart/form-data; boundary=------------------------d74496d66958873e

[Request Body]
--------------------------d74496d66958873e\r\n                ← libcurl 生成的边界
Content-Disposition: form-data; name="username"\r\n
\r\n
john_doe\r\n
--------------------------d74496d66958873e\r\n
Content-Disposition: form-data; name="avatar"; filename="a.png"\r\n
Content-Type: image/png\r\n
\r\n
[二进制图片数据]\r\n
--------------------------d74496d66958873e--\r\n              ← 结束边界
```

---

## 5. 文本字段 vs 文件字段：拼装细节对比

### 5.1 文本字段拼装

| 组件 | 内容 | 说明 |
|------|------|------|
| **边界行** | `--{boundary}\r\n` | 每个字段开始 |
| **Content-Disposition** | `form-data; name="字段名"` | 必填 |
| **Content-Type** | 无 | 文本字段通常不需要 |
| **空行** | `\r\n` | 分隔头和内容 |
| **内容** | 文本字符串 | UTF-8 编码 |
| **结尾** | `\r\n` | 字段结束 |

### 5.2 文件字段拼装

| 组件 | 内容 | 说明 |
|------|------|------|
| **边界行** | `--{boundary}\r\n` | 与文本字段相同 |
| **Content-Disposition** | `form-data; name="字段名"; filename="文件名"` | 额外的 filename 参数 |
| **Content-Type** | `MIME 类型` | 如 `image/png`、`application/pdf` 等 |
| **空行** | `\r\n` | 分隔头和内容 |
| **内容** | 二进制数据 | 原始字节，不编码 |
| **结尾** | `\r\n` | 字段结束 |

### 5.3 字段顺序保证

两种路径都通过特定机制保证字段顺序：

1. **浏览器路径**：`FormData.entries()` 按 `append()` 顺序迭代
2. **Desktop 路径**：使用 `Map` 存储（ES2015 保证插入顺序），序列化后传给 Rust

**排序规则**（EffectiveURL.ts:312-316）：
```typescript
// 文本字段在前，文件字段在后
arraySort((a, b) => {
  if (a.isFile) return 1    // 文件排后面
  if (b.isFile) return -1   // 非文件排前面
  return 0
})
```

---

## 6. 两条路径对比总结

| 对比项 | 浏览器 Fetch 路径 | Desktop Relay/libcurl 路径 |
|--------|------------------|---------------------------|
| **边界生成者** | 浏览器内置 HTTP 栈 | libcurl 库内部 |
| **生成时机** | 发送请求瞬间 | curl_easy_perform() 执行时 |
| **拼装位置** | JavaScript 运行时 + 浏览器内核 | Rust 后端 + libcurl |
| **数据传输** | 内存中 FormData 对象 | IPC 序列化的结构化数据 |
| **Content-Type 头** | 浏览器自动设置 | libcurl 自动设置 |
| **字段顺序** | FormData append 顺序 | Map 插入顺序（预处理时排序） |
| **二进制处理** | 浏览器直接处理 Blob | 转换为 Uint8Array 跨进程传输 |
| **适用场景** | Web 版 Hoppscotch | 桌面版 Hoppscotch |

### 6.1 共同点

1. **都不手动拼装报文**：两种路径都依赖底层库（浏览器/libcurl）处理边界生成和报文拼装
2. **字段排序相同**：预处理时都将文本字段排在前面，文件字段在后面
3. **最终报文格式一致**：都符合 RFC 7578 规范，服务器端无法区分来源

### 6.2 关键差异

- **边界值可见性**：两种路径下，JavaScript 层都**无法**获取或控制实际的 boundary 值
- **调试难度**：浏览器路径可通过 DevTools Network 面板查看完整报文；Desktop 路径需要启用 curl 调试日志
- **性能特征**：大文件上传时，Desktop 路径的 IPC 传输可能成为瓶颈

---

## 7. 边界值（Boundary）深入解析

### 7.1 边界值格式规范

根据 RFC 2046 Section 5.1.1：
```
boundary := 0*69<bchars> bcharsnospace
bchars := bcharsnospace / " "
bcharsnospace := DIGIT / ALPHA / "'" / "(" / ")" / "+" / "_" / "," / "-" / "." / "/" / ":" / "=" / "?"
```

### 7.2 典型边界值示例

| 生成源 | 示例 |
|--------|------|
| **Chrome/WebKit** | `----WebKitFormBoundary7MA4YWxkTrZu0gW` |
| **Firefox** | `---------------------------19813753121051141521880229351` |
| **libcurl** | `------------------------d74496d66958873e` |
| **Node.js form-data** | `--------------------------${随机串}` |

### 7.3 边界值唯一性保证

- 浏览器和 libcurl 都使用随机数生成器确保边界值不会出现在内容中
- 理论上存在冲突概率，但实际应用中可忽略
- 如果内容恰好包含边界字符串，会导致解析错误（极罕见）

---

## 8. CRLF 与换行处理

### 8.1 规范要求（RFC 7578 Section 4.1）

> The parts are separated by the boundary delimiter line. Each part is preceded by a boundary delimiter line, and the last part is followed by a closing boundary delimiter line. Each boundary delimiter line must be followed immediately by a CRLF.

### 8.2 拼装时的 CRLF 插入点

```
--{boundary}\r\n           ← 边界后必须有 CRLF
头字段\r\n                 ← 每个头字段后有 CRLF
\r\n                       ← 头结束后有空行（额外的 CRLF）
内容数据                   ← 内容本身
\r\n                       ← 内容后有 CRLF
--{boundary}--\r\n         ← 结束边界后有 CRLF
```

### 8.3 常见坑点

1. **缺失结尾 CRLF**：某些服务器对格式要求严格，缺少会导致解析失败
2. **LF 代替 CRLF**：Unix 风格换行在 multipart 中是错误的
3. **多余空行**：边界之间的多余空行可能被当作内容的一部分

---

## 9. 相关代码文件索引

| 文件路径 | 职责 |
|----------|------|
| `packages/hoppscotch-data/src/rest/v/9/body.ts` | 表单数据模型定义 |
| `packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts` | 请求体预处理、排序、环境变量解析 |
| `packages/hoppscotch-common/src/helpers/functional/formData.ts` | FormData 对象构建 |
| `packages/hoppscotch-kernel/src/relay/v/1.ts` | FormData 序列化、跨平台传输格式 |
| `packages/hoppscotch-kernel/src/relay/impl/web/v/1.ts` | 浏览器/Axios 路径实现 |
| `packages/hoppscotch-kernel/src/relay/impl/desktop/v/1.ts` | Desktop Relay 入口 |
| `packages/hoppscotch-desktop/plugin-workspace/relay/src/content.rs` | Rust 侧内容处理、libcurl Form 构建 |
| `packages/hoppscotch-desktop/plugin-workspace/relay/src/interop.rs` | Rust 侧数据结构定义 |

---

## 10. 参考规范

- **RFC 7578**：Returning Values from Forms: multipart/form-data
- **RFC 2046**：Multipurpose Internet Mail Extensions (MIME) Part Two: Media Types
- **RFC 2388**：Returning Values from Forms: multipart/form-data（已被 RFC 7578 取代）
- **HTML 5.2 Section 4.10.21.8**：Multipart form data
