# Multipart 请求体拼装逻辑分析

## 概述

Multipart/form-data 是 HTTP 协议中用于上传文件和表单数据的标准格式。Hoppscotch 项目中对 multipart 请求体的处理涉及多个模块协同工作，包括前端数据收集、序列化转换、Rust 后端拼装等环节。

---

## 1. 数据结构定义

### 1.1 前端数据模型 (`@hoppscotch/data`)

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
      z.object({
        isFile: z.literal(true),
        value: z.array(z.instanceof(Blob).nullable()).catch([]),
      }),
      z.object({
        isFile: z.literal(false),
        value: z.string(),
      }),
    ])
  )
```

**关键点**:
- 每个表单字段都有 `key`（字段名）、`active`（是否启用）、`contentType`（可选内容类型）
- 区分文件字段（`isFile: true`）和文本字段（`isFile: false`）
- 文件字段存储 `Blob[]` 数组，支持多文件上传
- 文本字段存储普通字符串

---

## 2. FormData 转换（前端浏览器环境）

### 2.1 基础转换函数

**文件**: `packages/hoppscotch-common/src/helpers/functional/formData.ts`

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
      // 有指定 contentType 时，包装为 Blob
      formData.append(
        key,
        new Blob([value], { type: contentType }),
        key  // 作为文件名
      )
      return
    }
    // 普通字段直接添加
    formData.append(key, value)
  })

  return formData
}
```

**工作原理**:
1. 利用浏览器原生 `FormData` API
2. 对于指定了 `contentType` 的字段，将值包装为 `Blob` 对象
3. 浏览器自动处理边界标识（boundary）的生成
4. 浏览器自动拼装请求体格式

---

## 3. 边界标识（Boundary）解析

### 3.1 边界标识检测与提取

**文件**: `packages/hoppscotch-common/src/helpers/curl/sub_helpers/contentParser.ts`

```typescript
const multipartFunctions = {
  // 从原始数据或 Content-Type 头获取边界标识
  getBoundary(rawData: string, rawContentType: string | undefined) {
    return pipe(
      rawContentType,
      O.fromNullable,
      O.filter((rct) => rct.length > 0),
      O.match(
        () => this.getBoundaryFromRawData(rawData),
        (rct) => this.getBoundaryFromRawContentType(rawData, rct)
      )
    )
  },

  // 直接从请求体数据中提取边界（通过正则匹配开头的 --XXXXX\r\n）
  getBoundaryFromRawData(rawData: string) {
    return pipe(
      rawData.match(/(-{2,}[A-Za-z0-9]+)\r\n/g),
      O.fromNullable,
      O.filter((boundaryMatch) => boundaryMatch.length > 0),
      O.map((matches) => matches[0].slice(0, -4))  // 去掉末尾的 \r\n
    )
  },

  // 从 Content-Type 头中解析 boundary 参数
  getBoundaryFromRawContentType(rawData: string, rawContentType: string) {
    return pipe(
      rawContentType.match(/boundary=(.+)/),
      O.fromNullable,
      O.filter((boundaryContentMatch) => boundaryContentMatch.length > 1),
      O.filter((matches) =>
        rawData.replaceAll("\r\n", "").endsWith("--" + matches[1] + "--")
      ),
      O.map((matches) => "--" + matches[1])  // 边界标识需要加上 -- 前缀
    )
  },
```

**边界标识格式规则**:
- Content-Type 头格式: `multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW`
- 实际请求体中边界前缀: `------WebKitFormBoundary7MA4YWxkTrZu0gW`
- 结束边界格式: `------WebKitFormBoundary7MA4YWxkTrZu0gW--`

### 3.2 使用边界分割请求体

```typescript
  // 使用边界标识分割 multipart 内容
  splitUsingBoundaryAndNewLines(rawData: string, boundary: string) {
    return pipe(
      rawData,
      S.split(RegExp(`${boundary}-*`)),  // 按边界分割
      RA.filter((p) => p !== "" && p.includes("name")),  // 过滤空片段
      RA.map((p) =>
        pipe(
          p.replaceAll(/\r\n+/g, "\r\n"),  // 规范化换行
          S.split("\r\n"),
          RA.filter((q) => q !== "")
        )
      )
    )
  },

  // 提取字段名和值
  getNameValuePair(pair: readonly string[]) {
    return pipe(
      pair,
      O.fromPredicate((p) => p.length > 1),
      O.chain((pair) => O.fromNullable(pair[0].match(/ name="(\w+)"/))),
      O.filter((nameMatch) => nameMatch.length > 0),
      O.chain((nameMatch) =>
        pipe(
          nameMatch[0],
          S.replace(/"/g, ""),
          S.split("="),
          O.fromPredicate((q) => q.length === 2),
          O.map(
            (nameArr) =>
              [nameArr[1], pair[0].includes("filename") ? "" : pair[1]] as [
                string,
                string,
              ]
          )
        )
      )
    )
  },
}
```

---

## 4. Kernel 内容转换层

### 4.1 ContentType 类型系统

**文件**: `packages/hoppscotch-kernel/src/relay/v/1.ts`

```typescript
export type FormDataValue =
  | { kind: "text"; value: string }
  | { kind: "file"; filename: string; contentType: string; data: Uint8Array }

export type ContentType =
  | { kind: "text"; content: string; mediaType: MediaType | string }
  | { kind: "multipart"; content: FormData; mediaType: MediaType | string }
  // ... 其他类型
```

### 4.2 FormData 序列化（跨平台传输）

```typescript
/**
 * 将浏览器 FormData 对象转换为可序列化的数组结构
 * 使用 Map 保持字段插入顺序（符合 RFC 7578 规范）
 */
const makeFormDataSerializable = async (
  formData: FormData
): Promise<[string, FormDataValue[]][]> => {
  const m = new Map<string, FormDataValue[]>()

  for (const [key, value] of formData.entries()) {
    if (value instanceof File || value instanceof Blob) {
      // 文件类型：读取二进制数据，提取文件名和类型
      const buffer = await value.arrayBuffer()
      const fileEntry: FormDataValue = {
        kind: "file",
        filename: value instanceof File ? value.name : "unknown",
        contentType: value.type || "application/octet-stream",
        data: new Uint8Array(buffer),
      }

      if (m.has(key)) {
        m.get(key)!.push(fileEntry)
      } else {
        m.set(key, [fileEntry])
      }
    } else {
      // 文本类型：直接转换为字符串
      const textEntry: FormDataValue = {
        kind: "text",
        value: value.toString(),
      }

      if (m.has(key)) {
        m.get(key)!.push(textEntry)
      } else {
        m.set(key, [textEntry])
      }
    }
  }

  return Array.from(m.entries())
}
```

**设计考量**:
- 使用 `Map` 而非普通对象保持插入顺序（ECMAScript 2015+ 规范）
- 文件转换为 `Uint8Array` 以支持跨进程/跨 VM 边界传输
- 支持同名字段多值（数组存储）
- 符合 RFC 7578 Section 5.2 关于字段顺序的建议

---

## 5. Rust 后端 - libcurl 拼装

### 5.1 内容处理器

**文件**: `packages/hoppscotch-desktop/plugin-workspace/relay/src/content.rs`

```rust
pub(crate) struct ContentHandler<'a> {
    handle: &'a mut Easy,      // curl easy handle
    headers: &'a mut HashMap<String, String>,
}

impl<'a> ContentHandler<'a> {
    #[tracing::instrument(skip(self), level = "debug")]
    pub(crate) fn set_content(&mut self, content: &ContentType) -> Result<()> {
        match content {
            ContentType::Multipart { content, media_type } => {
                tracing::info!(field_count = content.len(), "Setting multipart content");
                self.set_multipart_content(content, media_type)
            }
            // ... 其他内容类型
        }
    }
```

### 5.2 Multipart 表单构建

```rust
    fn set_form_content(
        &mut self,
        content: &Vec<(String, Vec<FormValue>)>,
        media_type: &MediaType,
    ) -> Result<()> {
        // 创建 curl Form 对象
        let mut form = curl::easy::Form::new();

        for (key, values) in content {
            for value in values {
                match value {
                    // 文本字段
                    FormValue::Text { value: text } => {
                        tracing::debug!(key = %key, text_length = text.len(), "Adding form text field");
                        form.part(key)
                            .contents(text.as_bytes())
                            .add()
                            .map_err(|e| {
                                // ... 错误处理
                            })?;
                    }
                    // 文件字段
                    FormValue::File {
                        filename,
                        content_type,
                        data,
                    } => {
                        tracing::debug!(
                            key = %key,
                            filename = %filename,
                            content_type = ?content_type,
                            data_length = data.len(),
                            "Adding form file field"
                        );
                        form.part(key)
                            .buffer(&filename, data.to_vec())  // 文件内容和名称
                            .content_type(&content_type.to_string())  // MIME 类型
                            .add()
                            .map_err(|e| {
                                // ... 错误处理
                            })?;
                    }
                }
            }
        }

        // 将构建好的表单绑定到 curl handle
        self.handle.httppost(form).map_err(|e| {
            // ... 错误处理
        })?;

        tracing::debug!("Form content set successfully");
        Ok(())
    }

    // multipart 复用 form 处理逻辑
    fn set_multipart_content(
        &mut self,
        content: &Vec<(String, Vec<FormValue>)>,
        media_type: &MediaType,
    ) -> Result<()> {
        self.set_form_content(content, media_type)
    }
```

**libcurl 自动处理的细节**:
1. **边界标识生成**: curl 自动生成唯一的 boundary 字符串
2. **Content-Disposition 头**: 自动添加 `form-data` 及 `name`、`filename` 参数
3. **Content-Type 头**: 为文件部分自动添加对应 MIME 类型
4. **换行格式**: 使用标准的 `\r\n` (CRLF) 换行
5. **结束边界**: 自动添加带 `--` 后缀的结束边界

---

## 6. 完整的 multipart 请求体格式示例

### 6.1 HTTP 请求示例

```http
POST /upload HTTP/1.1
Host: example.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Length: 345

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="username"

john_doe
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="avatar"; filename="avatar.png"
Content-Type: image/png

[二进制图片数据]
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

### 6.2 格式结构解析

```
边界标识行
  ├── -- + boundary
  └── \r\n

每个部分的结构:
  ├── Content-Disposition: form-data; name="字段名" [; filename="文件名"]
  ├── [Content-Type: MIME类型]  (仅文件)
  ├── \r\n (空行分隔头和内容)
  ├── 内容数据
  └── \r\n

结束边界:
  ├── -- + boundary + --
  └── \r\n
```

---

## 7. 数据流全景图

```
用户在 UI 输入表单数据
        ↓
[BodyParameters.vue] 收集字段
  ├── 文本字段: key + value
  └── 文件字段: key + Blob
        ↓
[transformContent] 转换为 ContentType
        ↓
浏览器环境: 使用原生 FormData API
        │
        ├─→ 浏览器 fetch: 浏览器自动拼装
        │
        └─→ Desktop App: 序列化传输
                ↓
        [makeFormDataSerializable]
                ↓
        转换为 [[key, [FormDataValue]]]
                ↓
        IPC 跨进程传输到 Rust
                ↓
        [ContentHandler.set_multipart_content]
                ↓
        curl::easy::Form 构建
                ↓
        libcurl 发送 HTTP 请求
```

---

## 8. 关键技术点总结

| 模块 | 职责 | 核心技术 |
|------|------|----------|
| **前端收集** | 用户输入、文件选择 | Vue 组件、Blob/File API |
| **边界解析** | 从原始数据提取 boundary | 正则表达式、RFC 7578 |
| **序列化** | 跨 VM/进程传输 | Uint8Array、Map 有序性 |
| **Rust 拼装** | 实际 HTTP 报文构建 | libcurl Form API |
| **内核抽象** | 统一内容类型接口 | ContentType tagged union |

### 重要规范引用
- **RFC 7578**: multipart/form-data 格式规范
- **RFC 2046**: MIME 多部分消息格式
- **ECMAScript 2015**: Map 插入顺序保证
