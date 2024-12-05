### 文件上传

Nest 提供了一个基于 [multer](https://github.com/expressjs/multer) 中间件包的内置模块来处理文件上传。Multer 处理以 `multipart/form-data` 格式发布的数据，这种格式主要用于通过 HTTP `POST` 请求上传文件。这个模块是完全可配置的，您可以根据应用程序的要求调整其行为。

> 警告 **警告** Multer 无法处理不支持的多部分格式（`multipart/form-data`）的数据。另外，请注意此包与 `FastifyAdapter` 不兼容。

为了更好的类型安全性，让我们安装 Multer 类型定义包：

```shell
$ npm i -D @types/multer
```

安装了这个包之后，我们现在可以使用 `Express.Multer.File` 类型（您可以如下导入此类型：`import { Express } from 'express'`）。

#### 基本示例

要上传单个文件，只需将 `FileInterceptor()` 拦截器绑定到路由处理器，并使用 `@UploadedFile()` 装饰器从 `request` 中提取 `file`。

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(@UploadedFile() file: Express.Multer.File) {
  console.log(file);
}
@@switch
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
@Bind(UploadedFile())
uploadFile(file) {
  console.log(file);
}
```

> 提示 **提示** `FileInterceptor()` 装饰器从 `@nestjs/platform-express` 包导出。`@UploadedFile()` 装饰器从 `@nestjs/common` 导出。

`FileInterceptor()` 装饰器接受两个参数：

- `fieldName`：字符串，提供 HTML 表单中包含文件的字段名称
- `options`：可选对象，类型为 `MulterOptions`。这与 multer 构造函数中使用的对象相同（更多详情[在这里](https://github.com/expressjs/multer#multeropts)）。

> 警告 **警告** `FileInterceptor()` 可能与第三方云提供商（如 Google Firebase 或其他）不兼容。

#### 文件验证

很多时候，验证传入的文件元数据（如文件大小或文件 mime 类型）非常有用。为此，您可以创建自己的 [Pipe](https://docs.nestjs.com/pipes) 并将其绑定到用 `UploadedFile` 装饰器注解的参数。以下示例演示了如何实现一个基本的文件大小验证管道：

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class FileSizeValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    // “value” 是一个包含文件属性和元数据的对象
    const oneKb = 1000;
    return value.size < oneKb;
  }
}
```

这可以与 `FileInterceptor` 结合使用，如下所示：

```typescript
@Post('file')
@UseInterceptors(FileInterceptor('file'))
uploadFileAndValidate(@UploadedFile(
  new FileSizeValidationPipe(),
  // 这里可以添加其他管道
) file: Express.Multer.File, ) {
  return file;
}
```

Nest 提供了一个内置管道来处理常见用例，并促进/标准化添加新用例。这个管道叫做 `ParseFilePipe`，您可以如下使用它：

```typescript
@Post('file')
uploadFileAndPassValidation(
  @Body() body: SampleDto,
  @UploadedFile(
    new ParseFilePipe({
      validators: [
        // ... 这里设置文件验证器实例集合
      ]
    })
  )
  file: Express.Multer.File,
) {
  return {
    body,
    file: file.buffer.toString(),
  };
}
```

如您所见，需要指定一个文件验证器数组，这些验证器将由 `ParseFilePipe` 执行。我们将讨论验证器的接口，但值得一提的是这个管道还有两个额外的 **可选** 选项：

<table>
  <tr>
    <td><code>errorHttpStatusCode</code></td>
    <td>如果任何验证器失败，则抛出的 HTTP 状态码。默认是 <code>400</code>（BAD REQUEST）</td>
  </tr>
  <tr>
    <td><code>exceptionFactory</code></td>
    <td>一个工厂，接收错误消息并返回一个错误。</td>
  </tr>
</table>

现在，回到 `FileValidator` 接口。要将验证器与此管道集成，您必须使用内置实现或提供您自己的自定义 `FileValidator`。以下示例：

```typescript
export abstract class FileValidator<TValidationOptions = Record<string, any>> {
  constructor(protected readonly validationOptions: TValidationOptions) {}

  /**
   * 表示根据构造函数中传递的选项，此文件是否应被视为有效。
   * @param file 请求对象中的文件
   */
  abstract isValid(file?: any): boolean | Promise<boolean>;

  /**
   * 如果验证失败，则构建错误消息。
   * @param file 请求对象中的文件
   */
  abstract buildErrorMessage(file: any): string;
}
```

> 提示 **提示** `FileValidator` 接口支持通过其 `isValid` 函数进行异步验证。为了利用类型安全，如果您使用的是 express（默认）作为驱动程序，也可以将 `file` 参数类型为 `Express.Multer.File`。

`FileValidator` 是一个常规类，它有权访问文件对象，并根据客户端提供的选项对其进行验证。Nest 提供了两个内置的 `FileValidator` 实现，您可以在项目中使用：

- `MaxFileSizeValidator` - 检查给定文件的大小是否小于提供的值（以 `bytes` 为单位）
- `FileTypeValidator` - 检查给定文件的 mime 类型是否与给定值匹配。

> 警告 **警告** 要验证文件类型，[FileTypeValidator](https://github.com/nestjs/nest/blob/master/packages/common/pipes/file/file-type.validator.ts) 类使用 multer 检测到的类型。默认情况下，multer 从用户设备上的文件扩展名派生文件类型。然而，它不检查实际的文件内容。由于文件可以重命名为任意扩展名，如果您的应用程序需要更安全的解决方案，请考虑使用自定义实现（例如检查文件的[魔术数字](https://www.ibm.com/support/pages/what-magic-number)）。

要了解如何将这些与上述的 `FileParsePipe` 结合使用，我们将使用最后呈现的示例的修改版本：

```typescript
@UploadedFile(
  new ParseFilePipe({
    validators: [
      new MaxFileSizeValidator({ maxSize: 1000 }),
      new FileTypeValidator({ fileType: 'image/jpeg' }),
    ],
  }),
)
file: Express.Multer.File,
```

> 提示 **提示** 如果验证器的数量大幅增加或它们的选项使文件变得混乱，您可以在单独的文件中定义此数组，并像 `fileValidators` 这样的命名常量一样导入到这里。

最后，您可以使用特殊的 `ParseFilePipeBuilder` 类来组合和构建您的验证器。如下所示使用它，您可以避免手动实例化每个验证器，而只需直接传递它们的选项：

```typescript
@UploadedFile(
  new ParseFilePipeBuilder()
    .addFileTypeValidator({
      fileType: 'jpeg',
    })
    .addMaxSizeValidator({
      maxSize: 1000
    })
    .build({
      errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY
    }),
)
file: Express.Multer.File,
```

> 提示 **提示** 默认情况下，文件存在是必需的，但您可以通过在 `build` 函数选项中添加 `fileIsRequired: false` 参数（与 `errorHttpStatusCode` 同级）使其成为可选。

#### 文件数组

要上传文件数组（用单个字段名称标识），使用 `FilesInterceptor()` 装饰器（注意装饰器名称中的复数 **Files**）。

此装饰器接受三个参数：

- `fieldName`：如上所述
- `maxCount`：可选数字，定义要接受的最大文件数量
- `options`：可选的 `MulterOptions` 对象，如上所述

使用 `FilesInterceptor()` 时，使用 `@UploadedFiles()` 装饰器从 `request` 中提取文件。

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
@@switch
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
@Bind(UploadedFiles())
uploadFile(files) {
  console.log(files);
}
```

> 提示 **提示** `FilesInterceptor()` 装饰器从 `@nestjs/platform-express` 包导出。`@UploadedFiles()` 装饰器从 `@nestjs/common` 导出。

#### 多个文件

要上传多个文件（所有文件都有不同的字段名键），使用 `FileFieldsInterceptor()` 装饰器。此装饰器接受两个参数：

- `uploadedFields`：一个对象数组，每个对象指定一个必需的 `name` 属性，其值为字符串，指定字段名称，如上所述，以及可选的 `maxCount` 属性，如上所述
- `options`：可选的 `MulterOptions` 对象，如上所述

使用 `FileFieldsInterceptor()` 时，使用 `@UploadedFiles()` 装饰器从 `request` 中提取文件。

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: 'background', maxCount: 1 },
]))
uploadFile(@UploadedFiles() files: { avatar?: Express.Multer.File[], background?: Express.Multer.File[] }) {
  console.log(files);
}
@@switch
@Post('upload')
@Bind(UploadedFiles())
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: 'background', maxCount: 1 },
]))
uploadFile(files) {
  console.log(files);
}
```

#### 任意文件

要上传所有字段，无论字段名键如何，使用 `AnyFilesInterceptor()` 装饰器。此装饰器可以接受如上所述的可选 `options` 对象。

使用 `AnyFilesInterceptor()` 时，使用 `@UploadedFiles()` 装饰器从 `request` 中提取文件。

```typescript
@@filename()
@Post('upload')
@UseInterceptors(AnyFilesInterceptor())
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
@@switch
@Post('upload')
@Bind(UploadedFiles())
@UseInterceptors(AnyFilesInterceptor())
uploadFile(files) {
  console.log(files);
}
```

#### 无文件

要接受 `multipart/form-data` 但不允许上传任何文件，使用 `NoFilesInterceptor`。这将把多部分数据设置为请求体的属性。如果请求中发送了任何文件，将抛出 `BadRequestException`。

```typescript
@Post('upload')
@UseInterceptors(NoFilesInterceptor())
handleMultiPartData(@Body() body) {
  console.log(body)
}
```

#### 默认选项

如上所述，可以在文件拦截器中指定 multer 选项。要设置默认选项，可以在导入 `MulterModule` 时调用静态 `register()` 方法，并传入支持的选项。您可以使用所有在这里列出的选项[这里](https://github.com/expressjs/multer#multeropts)。

```typescript
MulterModule.register({
  dest: './upload',
});
```

> 提示 **提示** `MulterModule` 类从 `@nestjs/platform-express` 包导出。

#### 异步配置

当您需要异步设置 `MulterModule` 选项而不是静态设置时，使用 `registerAsync()` 方法。与大多数动态模块一样，Nest 提供了几种技术来处理异步配置。

一种技术是使用工厂函数：

```typescript
MulterModule.registerAsync({
  useFactory: () => ({
    dest: './upload',
  }),
});
```

像其他[工厂提供者](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)一样，我们的工厂函数可以是 `async` 的，并且可以通过 `inject` 注入依赖项。

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    dest: configService.getString('MULTER_DEST'),
  }),
  inject: [ConfigService],
});
```

或者，您可以使用类而不是工厂函数来配置 `MulterModule`，如下所示：

```typescript
MulterModule.registerAsync({
  useClass: MulterConfigService,
});
```

上述构造在 `MulterModule` 内部实例化 `MulterConfigService`，使用它来创建所需的选项对象。请注意，在这种情况下，`MulterConfigService` 必须实现 `MulterOptionsFactory` 接口，如下所示。`MulterModule` 将在提供的类的实例对象上调用 `createMulterOptions()` 方法。

```typescript
@Injectable()
class MulterConfigService implements MulterOptionsFactory {
  createMulterOptions(): MulterModuleOptions {
    return {
      dest: './upload',
    };
  }
}
```

如果您想重用现有的选项提供者而不是在 `MulterModule` 内创建私有副本，请使用 `useExisting` 语法。

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

您还可以将所谓的 `extraProviders` 传递给 `registerAsync()` 方法。这些提供者将与模块提供者合并。

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useClass: ConfigService,
  extraProviders: [MyAdditionalProvider],
});
```

当您想要为工厂函数或类构造函数提供额外的依赖项时，这很有用。

#### 示例

一个工作示例可以在[这里](https://github.com/nestjs/nest/tree/master/sample/29-file-upload)找到。