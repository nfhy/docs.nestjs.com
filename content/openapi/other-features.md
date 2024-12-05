### 其他特性

这个页面列出了所有其他可能有用的特性。

#### 全局前缀

要忽略通过 `setGlobalPrefix()` 设置的全局前缀，使用 `ignoreGlobalPrefix`：

```typescript
const document = SwaggerModule.createDocument(app, options, {
  ignoreGlobalPrefix: true,
});
```

#### 全局参数

你可以使用 `DocumentBuilder` 为所有路由添加参数定义：

```typescript
const options = new DocumentBuilder().addGlobalParameters({
  name: 'tenantId',
  in: 'header',
});
```

#### 多个规范

`SwaggerModule` 提供了支持多个规范的方法。换句话说，你可以在不同的端点上提供不同的文档，具有不同的用户界面。

要支持多个规范，你的应用程序必须采用模块化方法编写。`createDocument()` 方法接受一个可选的第三个参数 `extraOptions`，这是一个包含名为 `include` 的属性的对象。`include` 属性接受一个值，该值是一个数组，包含你想要包含在该 Swagger 规范中的模块。

你可以如下设置多个规范支持：

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { CatsModule } from './cats/cats.module';
import { DogsModule } from './dogs/dogs.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  /**
   * createDocument(application, configurationOptions, extraOptions);
   *
   * createDocument 方法接受一个可选的第三个参数 "extraOptions"
   * 这是一个对象，具有 "include" 属性，你可以在这里传递一个数组
   * 包含你想要包含在该 Swagger 规范中的模块
   * 例如：CatsModule 和 DogsModule 将有两个独立的 Swagger 规范
   * 它们将在两个不同的 SwaggerUI 上公开，并且有两个不同的端点。
   */

  const options = new DocumentBuilder()
    .setTitle('Cats example')
    .setDescription('The cats API description')
    .setVersion('1.0')
    .addTag('cats')
    .build();

  const catDocumentFactory = () =>
    SwaggerModule.createDocument(app, options, {
      include: [CatsModule],
    });
  SwaggerModule.setup('api/cats', app, catDocumentFactory);

  const secondOptions = new DocumentBuilder()
    .setTitle('Dogs example')
    .setDescription('The dogs API description')
    .setVersion('1.0')
    .addTag('dogs')
    .build();

  const dogDocumentFactory = () =>
    SwaggerModule.createDocument(app, secondOptions, {
      include: [DogsModule],
    });
  SwaggerModule.setup('api/dogs', app, dogDocumentFactory);

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

现在你可以使用以下命令启动服务器：

```bash
$ npm run start
```

导航到 `http://localhost:3000/api/cats` 查看 Cats 的 Swagger UI：

<figure><img src="/assets/swagger-cats.png" /></figure>

相应地，`http://localhost:3000/api/dogs` 将公开 Dogs 的 Swagger UI：

<figure><img src="/assets/swagger-dogs.png" /></figure>

#### 探索栏中的下拉菜单

要在探索栏的下拉菜单中支持多个规范，你需要设置 `explorer: true` 并在你的 `SwaggerCustomOptions` 中配置 `swaggerOptions.urls`。

> 信息 **提示** 确保 `swaggerOptions.urls` 指向你的 Swagger 文档的 JSON 格式！要指定 JSON 文档，请在 `SwaggerCustomOptions` 中使用 `jsonDocumentUrl`。更多设置选项，请查看 [这里](/openapi/introduction#setup-options)。

以下是如何在探索栏的下拉菜单中设置多个规范的方法：

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { CatsModule } from './cats/cats.module';
import { DogsModule } from './dogs/dogs.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 主 API 选项
  const options = new DocumentBuilder()
    .setTitle('Multiple Specifications Example')
    .setDescription('Description for multiple specifications')
    .setVersion('1.0')
    .build();

  // 创建主 API 文档
  const document = SwaggerModule.createDocument(app, options);

  // 设置主 API Swagger UI，支持下拉菜单
  SwaggerModule.setup('api', app, document, {
    explorer: true,
    swaggerOptions: {
      urls: [
        {
          name: '1. API',
          url: 'api/swagger.json',
        },
        {
          name: '2. Cats API',
          url: 'api/cats/swagger.json',
        },
        {
          name: '3. Dogs API',
          url: 'api/dogs/swagger.json',
        },
      ],
    },
    jsonDocumentUrl: '/api/swagger.json',
  });

  // Cats API 选项
  const catOptions = new DocumentBuilder()
    .setTitle('Cats Example')
    .setDescription('Description for the Cats API')
    .setVersion('1.0')
    .addTag('cats')
    .build();

  // 创建 Cats API 文档
  const catDocument = SwaggerModule.createDocument(app, catOptions, {
    include: [CatsModule],
  });

  // 设置 Cats API Swagger UI
  SwaggerModule.setup('api/cats', app, catDocument, {
    jsonDocumentUrl: '/api/cats/swagger.json',
  });

  // Dogs API 选项
  const dogOptions = new DocumentBuilder()
    .setTitle('Dogs Example')
    .setDescription('Description for the Dogs API')
    .setVersion('1.0')
    .addTag('dogs')
    .build();

  // 创建 Dogs API 文档
  const dogDocument = SwaggerModule.createDocument(app, dogOptions, {
    include: [DogsModule],
  });

  // 设置 Dogs API Swagger UI
  SwaggerModule.setup('api/dogs', app, dogDocument, {
    jsonDocumentUrl: '/api/dogs/swagger.json',
  });

  await app.listen(3000);
}

bootstrap();
```

在这个例子中，我们设置了一个主 API 以及 Cats 和 Dogs 的独立规范，每个都可以从探索栏的下拉菜单中访问。