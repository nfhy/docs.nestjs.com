### 引言

[OpenAPI](https://swagger.io/specification/) 规范是一种用于描述 RESTful API 的语言无关定义格式。Nest 提供了一个专门的[模块](https://github.com/nestjs/swagger)，允许通过使用装饰器生成此类规范。

#### 安装

要开始使用它，我们首先安装所需的依赖项。

```bash
$ npm install --save @nestjs/swagger
```

#### 启动

安装过程完成后，打开 `main.ts` 文件并使用 `SwaggerModule` 类初始化 Swagger：

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Cats example')
    .setDescription('The cats API description')
    .setVersion('1.0')
    .addTag('cats')
    .build();
  const documentFactory = () => SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, documentFactory);

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> 信息 **提示** 工厂方法 `SwaggerModule#createDocument()` 专门用于在请求时生成 Swagger 文档。这种方法有助于节省一些初始化时间，生成的文档是一个符合 [OpenAPI 文档](https://swagger.io/specification/#openapi-document) 规范的可序列化对象。您也可以不通过 HTTP 服务文档，而是将其保存为 JSON 或 YAML 文件，并以各种方式使用。

`DocumentBuilder` 有助于构建符合 OpenAPI 规范的基础文档。它提供了几种方法，允许设置标题、描述、版本等属性。为了创建一个完整的文档（包含所有 HTTP 路由定义），我们使用 `SwaggerModule` 类的 `createDocument()` 方法。此方法接受两个参数，一个应用程序实例和一个 Swagger 选项对象。或者，我们可以提供一个第三参数，该参数应该是 `SwaggerDocumentOptions` 类型。更多内容请参见 [文档选项部分](/openapi/introduction#document-options)。

创建文档后，我们可以调用 `setup()` 方法。它接受：

1. 挂载 Swagger UI 的路径
2. 应用程序实例
3. 上面实例化的文档对象
4. 可选配置参数（更多信息请参见[这里](/openapi/introduction#setup-options)）

现在您可以运行以下命令启动 HTTP 服务器：

```bash
$ npm run start
```

在应用程序运行时，打开浏览器并导航到 `http://localhost:3000/api`。您应该看到 Swagger UI。

<figure><img src="/assets/swagger1.png" /></figure>

如您所见，`SwaggerModule` 自动反映您所有的端点。

> 信息 **提示** 要生成并下载 Swagger JSON 文件，请导航到 `http://localhost:3000/api-json`（假设您的 Swagger 文档可在 `http://localhost:3000/api` 下获得）。
> 您还可以使用 `@nestjs/swagger` 的 `setup` 方法仅暴露在您选择的路由上，如下所示：

> ```typescript
> SwaggerModule.setup('swagger', app, document, {
>   jsonDocumentUrl: 'swagger/json',
> });
> ```

> 这将在 `http://localhost:3000/swagger/json` 上暴露它。

> 警告 **警告** 当使用 `fastify` 和 `helmet` 时，可能会遇到 [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的问题，要解决此冲突，请按如下方式配置 CSP：

> ```typescript
> app.register(helmet, {
>   contentSecurityPolicy: {
>     directives: {
>       defaultSrc: ['self'],
>       styleSrc: ['self', 'unsafe-inline'],
>       imgSrc: ['self', 'data:', 'validator.swagger.io'],
>       scriptSrc: ['self', "https: 'unsafe-inline'"],
>     },
>   },
> });

> // 如果您根本不打算使用 CSP，可以使用这个：
> app.register(helmet, {
>   contentSecurityPolicy: false,
> });
> ```

#### 文档选项

创建文档时，可以提供一些额外选项来微调库的行为。这些选项应该是 `SwaggerDocumentOptions` 类型，可以是以下内容：

```TypeScript
export interface SwaggerDocumentOptions {
  /**
   * 包含在规范中的模块列表
   */
  include?: Function[];

  /**
   * 应该检查并包含在规范中的附加模型
   */
  extraModels?: Function[];

  /**
   * 如果为 `true`，则 swagger 将忽略通过 `setGlobalPrefix()` 方法设置的全局前缀
   */
  ignoreGlobalPrefix?: boolean;

  /**
   * 如果为 `true`，则 swagger 还将从 `include` 模块导入的模块中加载路由
   */
  deepScanRoutes?: boolean;

  /**
   * 自定义 operationIdFactory，将用于基于 `controllerKey`、`methodKey` 和版本生成 `operationId`。
   * @default () => controllerKey_methodKey_version
   */
  operationIdFactory?: OperationIdFactory;

  /**
   * 自定义 linkNameFactory，将用于在响应的 `links` 字段中生成链接的名称
   *
   * @see [Link objects](https://swagger.io/docs/specification/links/)
   *
   * @default () => `${controllerKey}_${methodKey}_from_${fieldKey}`
   */
  linkNameFactory?: (
    controllerKey: string,
    methodKey: string,
    fieldKey: string
  ) => string;

  /**
   * 根据控制器名称自动生成标签。
   * 如果为 `false`，则必须使用 `@ApiTags()` 装饰器定义标签。
   * 否则，将使用没有 `Controller` 后缀的控制器名称。
   * @default true
   */
  autoTagControllers?: boolean;
}
```

例如，如果您想确保库生成的操作名称像 `createUser` 而不是 `UserController_createUser`，您可以设置如下：

```TypeScript
const options: SwaggerDocumentOptions =  {
  operationIdFactory: (
    controllerKey: string,
    methodKey: string
  ) => methodKey
};
const documentFactory = () => SwaggerModule.createDocument(app, config, options);
```

#### 设置选项

您可以通过将满足 `SwaggerCustomOptions` 接口的选项对象作为第四个参数传递给 `SwaggerModule#setup` 方法来配置 Swagger UI。

```TypeScript
export interface SwaggerCustomOptions {
  /**
   * 如果为 `true`，则 Swagger 资源路径将通过 `setGlobalPrefix()` 设置的全局前缀进行前缀处理。
   * 默认：`false`。
   * @see https://docs.nestjs.com/faq/global-prefix
   */
  useGlobalPrefix?: boolean;

  /**
   * 如果为 `false`，则仅提供 API 定义（JSON 和 YAML）(在 `/{path}-json` 和 `/{path}-yaml` 上)。
   * 如果您已经在其他地方托管了 Swagger UI，并且只想提供 API 定义，这特别有用。
   * 默认：`true`。
   */
  swaggerUiEnabled?: boolean;

  /**
   * Swagger UI 加载 API 定义的 URL 点。
   */
  swaggerUrl?: string;

  /**
   * 提供 JSON API 定义的路径。
   * 默认：`<path>-json`。
   */
  jsonDocumentUrl?: string;

  /**
   * 提供 YAML API 定义的路径。
   * 默认：`<path>-yaml`。
   */
  yamlDocumentUrl?: string;

  /**
   * 在提供 OpenAPI 文档之前允许修改它的钩子。
   * 文档生成后，在作为 JSON & YAML 提供之前被调用。
   */
  patchDocumentOnRequest?: <TRequest = any, TResponse = any>(
    req: TRequest,
    res: TResponse,
    document: OpenAPIObject
  ) => OpenAPIObject;

  /**
   * 如果为 `true`，则在 Swagger UI 界面中显示 OpenAPI 定义的选择器。
   * 默认：`false`。
   */
  explorer?: boolean;

  /**
   * 额外的 Swagger UI 选项
   */
  swaggerOptions?: SwaggerUiOptions;

  /**
   * 注入到 Swagger UI 页面的自定义 CSS 样式。
   */
  customCss?: string;

  /**
   * 在 Swagger UI 页面加载的自定义 CSS 样式表的 URL。
   */
  customCssUrl?: string | string[];

  /**
   * 在 Swagger UI 页面加载的自定义 JavaScript 文件的 URL。
   */
  customJs?: string | string[];

  /**
   * 在 Swagger UI 页面加载的自定义 JavaScript 脚本。
   */
  customJsStr?: string | string[];

  /**
   * Swagger UI 页面的自定义 favicon。
   */
  customfavIcon?: string;

  /**
   * Swagger UI 页面的自定义标题。
   */
  customSiteTitle?: string;

  /**
   * 包含静态 Swagger UI 资产的文件系统路径（例如：./node_modules/swagger-ui-dist）。
   */
  customSwaggerUiPath?: string;

  /**
   * @deprecated 此属性无效果。
   */
  validatorUrl?: string;

  /**
   * @deprecated 此属性无效果。
   */
  url?: string;

  /**
   * @deprecated 此属性无效果。
   */
  urls?: Record<'url' | 'name', string[]>;
}
```

#### 示例

一个工作示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/11-swagger) 找到。