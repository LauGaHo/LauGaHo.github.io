# JS 二进制 File, Blob, FileReader, ArrayBuffer, Base64

JavaScript 提供了一些 API 来处理文件或者原始文件数据，例如：File, Blob, FileReader, ArrayBuffer, Base64 等。看看他们是如何使用，他们之间有什么区别和联系。

## Blob

Blob 全称为 Binary Large Object，即二进制大对象，它是 JavaScript 中的一个对象，表示原始的类似文件的数据。MDN 对 Blob 的理解如下：

> Blob 对象表示一个不可变，原始的类文件对象。它的数据可以按文本或二进制的格式进行读取，也可以转成 ReadableStream 来用于数据操作。

实际上，Blob 对象是包含只读原始数据的类文件对象。简单来说，Blob 对象就是一个不可修改的二进制文件。

### 创建 Blob

可以使用 `Blob()` 构造函数来创建一个 Blob 对象:

```typescript
new Blob(array, options);
```

该构造函数有两个参数：

1. `array`: 由 `ArrayBuffer`, `ArrayBufferView`, `Blob`, `DOMString` 等对象构成的，将会被放进 `Blob`。

2. `options`: 可选的 `BlobPropertyBag` 字典，它可能会指定如下两个属性

   1. `type`: 默认值为 `''`，表示将会被放入到 `blob` 中的数组内容的 MIME 类型。

   2. `endings`: 默认值为 `"transparent"`，用于指定包含行结束符 `\n` 的字符串如何被写入，不常用。

常见的 MIME 类型如下所示：

| MIME 类型        | 描述       |
| ---------------- | ---------- |
| text/plain       | 纯文本文档 |
| text/html        | HTML       |
| text/javascript  | JavaScript |
| text/css         | CSS        |
| application/json | JSON       |
| application/pdf  | PDF        |
| application/xml  | XML        |
| image/jpeg       | JPEG       |
| image/png        | PNG        |
| image/gif        | GIF        |
| image/svg-xml    | SVG        |
| audio/mpeg       | MP3        |
| video/mpeg       | MP4        |

可以看一下简单的例子：

```typescript
const blob = new Blob(["hello world"], { type: "text/plain" });
```

上方的这段代码是创建了一个类似文件的对象。在这个 blob 对象上有两个属性：

1. `size`: Blob 对象中所包含数据的大小（单位字节）。

2. `type`: 字符串，认为该 blob 对象所包含的 MIME 类型。如果类型未知，则为空字符串。

下边看一下对应打印的结果：

```typescript
const blob = new Blob(["hello world"], { type: "text/plain" });

console.log(blob.size); // 11
console.log(blob.type); // "text/plain"
```

> 注意，字符串 "Hello World" 是使用 UTF-8 编码的，因此它的每个字符占用 1 个字节。

### Blob 分片

除了使用 `Blob()` 构造函数来创建 Blob 对象之外，还可以从 blob 对象中创建 Blob 对象，也就是将 blob 对象切片。Blob 对象内置了 `slice()` 方法用来将 blob 对象分片，其语法如下：

```typescript
const blob = instanceOfBlob.slice(start, end, contentType);
```

其中有三个参数：

1. `start`: 设置切片的起点，即切片开始位置。默认值为 0，这意味着切片应该从第一个字节开始。

2. `end`: 设置切片的结束点，会对该位置之前的数据进行切片。默认值为 `blob.size`。

3. `contentType`: 设置新 blob 的 MIME 类型，如果省略了，则默认为 `blob` 的原始值。

```typescript
const iframe = document.getElementByTagName("iframe")[0];

const blob = new Blob(["hello world"], { type: "text/plain" });

const subBlob = blob.slice(0, 5);

iframe.src = URL.createObjectURL(subBlob);
```

此时页面会显示 "Hello"。

## File

文件 File 接口提供有关文件的信息，并允许网页中的 JavaScript 访问其内容，实际上，File 对象是特殊类型的 Blob，且可以用在任意的 Blob 类型的 context 中，Blob 的属性和方法都可以用于 File 对象。

> 注意：File 对象中只存在于浏览器环境中，在 Node.js 环境中不存在。

在 JavaScript 中，主要有一种方式来获取 File 对象：

1. `<input>` 元素上选择文件后返回的 `FileList` 对象。

### input

首先定义一个输入类型为 `file` 的 `input` 标签：

```html
<input type="file" id="fileInput" multiple="multiple" />
```

这里给 `input`标签页添加了三个属性：

1. `type="file"`: 指定 `input` 的输入属性为文件。

2. `id="fileInput"`: 指定 `input` 的唯一 id。

3. `multiple="multiple"`: 指定 `input` 可以同时上传多个文件。

下面给 `input` 标签添加 `onchange` 事件，当选择文件并上传之后触发：

```typescript
const fileInput = document.getElementById("fileInput");

fileInput.onchange = (e) => {
  console.log(e.target.files);
};
```

当点击上传文件时，控制台就会输出一个 `FileList` 数组，这个数组的每个元素都是一个 `File` 对象，一个上传的文件就对应一个 `File` 对象。

每个 `File` 对象都包含文件的一些属性，这些属性都继承自 `Blob` 对象：

- `lastModified`: 引用文件最后修改日期，为自 1970 年 1 月 1 日 0:00 以来的毫秒数。

- `lastModifiedDate`: 引用文件的最后修改日期。

- `name`: 引用文件的文件名。

- `size`: 引用文件的文件大小。

- `type`: 文件的媒体类型 MIME。

- `webkitRelativePath`: 文件的路径或 URL。

通常在上传文件的时候，可以通过对比 `size` 属性来限制文件大小，通过比对 `type` 来限制上传文件的格式等。

## FileReader

`FileReader` 是一个异步 API，用于读取文件并提取其内容以供进一步使用。`FileReader` 可以将 `Blob` 读取为不同的格式。

> 注意：FileReader 仅用于安全的方式从用户（远程）系统读取文件内容，不能用于从文件系统中按路径名简单地读取文件。

### 基本使用

可以使用 `FileReader` 构造函数来创建一个 `FileReader` 对象：

```typescript
const reader = new FileReader();
```

这个对象常用属性如下：

1. `error`: 表示在读取文件时发生的错误。

2. `result`: 表示文件内容，该属性仅在读取操作完成后才有效，数据的格式取决于使用哪个方法来启动读取操作。

3. `readyState`: 表示 `FileReader` 状态的数字。取值如下：

| 常量名  | 值  | 描述                 |
| ------- | --- | -------------------- |
| EMPTY   | 0   | 还没加载任何数据     |
| LOADING | 1   | 数据正在被加载       |
| DONE    | 2   | 已完成全部的读取请求 |

`FileReader` 对象提供了以下方法来加载文件：
