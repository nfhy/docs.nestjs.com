### 流式传输文件

> **注意** 本章节展示了如何从您的**HTTP应用程序**中流式传输文件。下面提供的示例不适用于GraphQL或微服务应用程序。

有时候，您可能希望从REST API向客户端发送文件。使用Nest进行此操作，通常您会执行以下操作：

```ts
@Controller('file')
export class FileController {
  @Get()
  getFile(@Res() res: Response) {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    file.pipe(res);
  }
}
```

但是这样做会导致您失去访问后控制器拦截器逻辑的机会。为了处理这个问题，您可以返回一个`StreamableFile`实例，框架将在内部处理响应的管道传输。

#### Streamable File类

`StreamableFile`是一个持有要返回的流的类。要创建一个新的`StreamableFile`，您可以将`Buffer`或`Stream`传递给`StreamableFile`构造函数。

> **提示** `StreamableFile`类可以从`@nestjs/common`导入。

#### 跨平台支持

Fastify默认支持发送文件，无需调用`stream.pipe(res)`，因此您根本不需要使用`StreamableFile`类。但是，Nest支持在两种平台类型中使用`StreamableFile`，所以如果您在Express和Fastify之间切换，就无需担心两个引擎之间的兼容性问题。

#### 示例

您可以在下面找到一个简单的示例，将`package.json`作为文件而不是JSON返回，但这个想法自然扩展到图片、文档和其他任何文件类型。

```ts
import { Controller, Get, StreamableFile } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }
}
```

默认的内容类型（`Content-Type` HTTP响应头的值）是`application/octet-stream`。如果您需要自定义这个值，可以使用`StreamableFile`的`type`选项，或者使用`res.set`方法或[`@Header()`](/controllers#headers)装饰器，如下所示：

```ts
import { Controller, Get, StreamableFile, Res } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';
import type { Response } from 'express'; // 假设我们使用的是ExpressJS HTTP适配器

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file, {
      type: 'application/json',
      disposition: 'attachment; filename="package.json"',
      // 如果您想将Content-Length值定义为文件长度以外的其他值：
      // length: 123,
    });
  }

  // 或者：
  @Get()
  getFileChangingResponseObjDirectly(@Res({ passthrough: true }) res: Response): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    res.set({
      'Content-Type': 'application/json',
      'Content-Disposition': 'attachment; filename="package.json"',
    });
    return new StreamableFile(file);
  }

  // 或者：
  @Get()
  @Header('Content-Type', 'application/json')
  @Header('Content-Disposition', 'attachment; filename="package.json"')
  getFileUsingStaticValues(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }
}
```