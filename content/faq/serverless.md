### 无服务器计算

无服务器计算是一种云计算执行模型，云服务提供商根据需求为客户分配机器资源，并代表客户管理服务器。当应用程序不在使用时，不会为应用程序分配计算资源。定价基于应用程序实际消耗的资源量（来源：[维基百科](https://en.wikipedia.org/wiki/Serverless_computing)）。

在**无服务器架构**中，您只需关注应用程序代码中的各个函数。AWS Lambda、Google Cloud Functions和Microsoft Azure Functions等服务负责所有物理硬件、虚拟机操作系统和Web服务器软件的管理。

> 信息提示：本章节不涉及无服务器函数的优缺点，也不深入任何云服务提供商的具体内容。

#### 冷启动

冷启动是指您的代码在一段时间内首次执行。根据您使用的云服务提供商，它可能涉及多种不同的操作，从下载代码和引导运行时环境，到最终运行您的代码。

这个过程增加了**显著的延迟**，这取决于几个因素，比如语言、应用程序需要的包的数量等。

冷启动很重要，尽管有些因素我们无法控制，但我们仍然可以在我们的方面做很多事情，使其尽可能短。

虽然您可以将Nest视为一个为复杂、企业级应用程序设计的完整框架，但它也**适用于更“简单”的应用程序**（或脚本）。例如，使用[独立应用程序](/standalone-applications)功能，您可以在简单的工作器、CRON作业、CLI或无服务器函数中利用Nest的DI系统。

#### 基准测试

为了更好地理解在无服务器函数的背景下使用Nest或其他知名库（如`express`）的成本，让我们比较Node运行时需要运行以下脚本的时间：

```typescript
// #1 Express
import * as express from 'express';

async function bootstrap() {
  const app = express();
  app.get('/', (req, res) => res.send('Hello world!'));
  await new Promise<void>((resolve) => app.listen(3000, resolve));
}
bootstrap();

// #2 Nest（带有@nestjs/platform-express）
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { logger: ['error'] });
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();

// #3 Nest作为独立应用程序（无HTTP服务器）
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AppService } from './app.service';

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule, {
    logger: ['error'],
  });
  console.log(app.get(AppService).getHello());
}
bootstrap();

// #4 原生Node.js脚本
async function bootstrap() {
  console.log('Hello world!');
}
bootstrap();
```

对于所有这些脚本，我们使用了`tsc`（TypeScript）编译器，因此代码保持未打包（未使用`webpack`）。

|                                      |                   |
| ------------------------------------ | ----------------- |
| Express                              | 0.0079s (7.9ms)   |
| Nest带有`@nestjs/platform-express` | 0.1974s (197.4ms) |
| Nest（独立应用程序）        | 0.1117s (111.7ms) |
| 原生Node.js脚本                   | 0.0071s (7.1ms)   |

> 信息提示：机器：MacBook Pro Mid 2014，2.5 GHz 四核Intel Core i7，16 GB 1600 MHz DDR3，SSD。

现在，让我们重复所有基准测试，但这次使用`webpack`（如果您安装了[Nest CLI](/cli/overview)，可以运行`nest build --webpack`）将我们的应用程序捆绑成一个单一的可执行JavaScript文件。

然而，我们不会使用Nest CLI自带的默认`webpack`配置，我们将确保捆绑所有依赖项（`node_modules`），如下所示：

```javascript
module.exports = (options, webpack) => {
  const lazyImports = [
    '@nestjs/microservices/microservices-module',
    '@nestjs/websockets/socket-module',
  ];

  return {
    ...options,
    externals: [],
    plugins: [
      ...options.plugins,
      new webpack.IgnorePlugin({
        checkResource(resource) {
          if (lazyImports.includes(resource)) {
            try {
              require.resolve(resource);
            } catch (err) {
              return true;
            }
          }
          return false;
        },
      }),
    ],
  };
};
```

> 信息提示：要指示Nest CLI使用此配置，请在项目的根目录中创建一个新的`webpack.config.js`文件。

使用此配置，我们得到了以下结果：

|                                      |                  |
| ------------------------------------ | ---------------- |
| Express                              | 0.0068s (6.8ms)  |
| Nest带有`@nestjs/platform-express` | 0.0815s (81.5ms) |
| Nest（独立应用程序）        | 0.0319s (31.9ms) |
| 原生Node.js脚本                   | 0.0066s (6.6ms)  |

> 信息提示：机器：MacBook Pro Mid 2014，2.5 GHz 四核Intel Core i7，16 GB 1600 MHz DDR3，SSD。

> 信息提示：您可以通过应用额外的代码压缩和优化技术（使用`webpack`插件等）进一步优化它。

如您所见，您编译（以及是否捆绑代码）的方式至关重要，对整体启动时间有重大影响。使用`webpack`，您可以将独立Nest应用程序（一个模块、控制器和服务的起始项目）的平均启动时间降低到大约32ms，对于常规的基于HTTP的NestJS应用程序，降低到大约81.5ms。

对于更复杂的Nest应用程序，例如，有10个资源（通过`$ nest g resource`模式生成=10个模块、10个控制器、10个服务、20个DTO类、50个HTTP端点+`AppModule`），在MacBook Pro Mid 2014，2.5 GHz 四核Intel Core i7，16 GB 1600 MHz DDR3，SSD上的总体启动时间大约为0.1298s（129.8ms）。将单体应用程序作为无服务器函数运行通常没有太多意义，因此将此基准测试更多地视为启动时间可能随着应用程序增长而增加的示例。

#### 运行时优化

到目前为止，我们已经介绍了编译时优化。这些与您在应用程序中定义提供者和加载Nest模块的方式无关，这在应用程序变大时起着重要作用。

例如，想象一下，数据库连接被定义为[异步提供者](/fundamentals/async-providers)。异步提供者旨在延迟应用程序启动，直到一个或多个异步任务完成。

这意味着，如果您的无服务器函数平均需要2秒才能连接到数据库（在引导时），您的端点将需要至少额外两秒钟（因为它必须等待连接建立）才能发送响应（当它是冷启动且您的应用程序尚未运行时）。

如您所见，在**无服务器环境中**，引导时间很重要，您在结构化提供者方面的方式与此有所不同。

另一个好例子是，如果您使用Redis进行缓存，但只在某些场景下使用。也许，在这种情况下，您不应该将Redis连接定义为异步提供者，因为它会减慢引导时间，即使它不是这次特定函数调用所必需的。

此外，有时您可以使用`LazyModuleLoader`类懒加载整个模块，如[本章](/fundamentals/lazy-loading-modules)所述。缓存是一个很好的例子。

想象一下，您的应用程序有一个`CacheModule`，它内部连接到Redis，并且还导出了`CacheService`与Redis存储进行交互。如果您不需要它用于所有潜在的函数调用，您可以只是在需要时按需加载它。这样，当冷启动发生时，您将获得更快的启动时间（当不需要缓存时的所有调用）。

```typescript
if (request.method === RequestMethod[RequestMethod.GET]) {
  const { CacheModule } = await import('./cache.module');
  const moduleRef = await this.lazyModuleLoader.load(() => CacheModule);

  const { CacheService } = await import('./cache.service');
  const cacheService = moduleRef.get(CacheService);

  return cacheService.get(ENDPOINT_KEY);
}
```

另一个好例子是Webhook或工作器，根据特定条件（例如，输入参数），可能会执行不同的操作。

在这种情况下，您可以在路由处理程序中指定一个条件，该条件为特定函数调用懒加载适当的模块，并仅懒加载每个其他模块。

```typescript
if (workerType === WorkerType.A) {
  const { WorkerAModule } = await import('./worker-a.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerAModule);
  // ...
} else if (workerType === WorkerType.B) {
  const { WorkerBModule } = await import('./worker-b.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerBModule);
  // ...
}
```

#### 示例集成

您的应用程序的入口文件（通常是`main.ts`文件）应该看起来**取决于几个因素**，因此**没有单一模板**适用于每种场景。

例如，启动您的无服务器函数所需的初始化文件因云服务提供商（AWS、Azure、GCP等）而异。此外，根据您的应用程序是运行具有多个路由/端点的典型HTTP应用程序还是仅提供单个路由（或执行特定代码部分），您的应用程序代码看起来会有所不同（例如，对于端点每个函数的方法，您可以使用`NestFactory.createApplicationContext`而不是启动整个HTTP服务器，设置中间件等）。

仅作示例，我们将Nest（使用`@nestjs/platform-express`，因此启动整个功能齐全的HTTP路由器）与[Serverless](https://www.serverless.com/)框架（在这种情况下，针对AWS Lambda）集成。如前所述，您的代码将根据您的选择的云服务提供商和许多其他因素而有所不同。

首先，让我们安装所需的包：

```bash
$ npm i @codegenie/serverless-express aws-lambda
$ npm i -D @types/aws-lambda serverless-offline
```

> 信息提示：为了加快开发周期，我们安装了`serverless-offline`插件，该插件模拟AWS λ和API Gateway。

安装过程完成后，让我们创建`serverless.yml`文件以配置Serverless框架：

```yaml
service: serverless-example

plugins:
  - serverless-offline

provider:
  name: aws
  runtime: nodejs14.x

functions:
  main:
    handler: dist/main.handler
    events:
      - http:
          method: ANY
          path: /
      - http:
          method: ANY
          path: '{proxy+}'
```

> 信息提示：要了解更多关于Serverless框架的信息，请访问[官方文档](https://www.serverless.com/framework/docs/)。

有了这些设置，我们现在可以导航到`main.ts`文件，并用所需的样板代码更新我们的引导代码：

```typescript
import { NestFactory } from '@nestjs/core';
import serverlessExpress from '@codegenie/serverless-express';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';

let server: Handler;

async function bootstrap(): Promise<Handler> {
  const app = await NestFactory.create(AppModule);
  await app.init();

  const expressApp = app.getHttpAdapter().getInstance();
  return serverlessExpress({ app: expressApp });
}

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  server = server ?? (await bootstrap());
  return server(event, context, callback);
};
```

> 信息提示：对于创建多个无服务器函数并在它们之间共享公共模块，我们建议使用[CLI Monorepo模式](/cli/monorepo#monorepo-mode)。

> 警告：如果您使用`@nestjs/swagger`包，需要额外的几个步骤才能使其在无服务器函数的上下文中正常工作。查看这个[线程](https://github.com/nestjs/swagger/issues/199)了解更多信息。

接下来，打开`tsconfig.json`文件，并确保启用`esModuleInterop`选项，以便正确加载`@codegenie/serverless-express`包。

```json
{
  "compilerOptions": {
    ...,
    "esModuleInterop": true
  }
}
```

现在我们可以构建我们的应用程序（使用`nest build`或`tsc`），并使用`serverless` CLI启动我们的lambda函数本地：

```bash
$ npm run build
$ npx serverless offline
```

一旦应用程序运行，打开您的浏览器并导航到`http://localhost:3000/dev/[ANY_ROUTE]`（其中`[ANY_ROUTE]`是您的应用程序中注册的任何端点）。

在上述部分中，我们已经展示了使用`webpack`和捆绑您的应用程序可以对整体启动时间产生重大影响。

然而，为了使它与我们的示例一起工作，您必须在`webpack.config.js`文件中添加一些额外的配置。通常，为了确保我们的`handler`函数被选中，我们必须将`output.libraryTarget`属性更改为`commonjs2`。

```javascript
return {
  ...options,
  externals: [],
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',
  },
  // ...其余配置
};
```

有了这个设置，您现在可以使用`$ nest build --webpack`编译您的函数代码（然后使用`$ npx serverless offline`进行测试）。

还建议（但**不是必需**，因为它会减慢您的构建过程）安装`terser-webpack-plugin`包，并覆盖其配置以在压缩生产构建时保持类名不变。不这样做可能会导致在应用程序中使用`class-validator`时行为不正确。

```javascript
const TerserPlugin = require('terser-webpack-plugin');

return {
  ...options,
  externals: [],
  optimization: {
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          keep_classnames: true,
        },
      }),
    ],
  },
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',
  },
  // ...其余配置
};
```

#### 使用独立应用程序功能

或者，如果您想保持您的函数非常轻量级，并且您不需要任何与HTTP相关的功能（路由，但也包括守卫、拦截器、管道等），您可以只使用`NestFactory.createApplicationContext`（如前所述）而不是运行整个HTTP服务器（以及底层的`express`），如下所示：

```typescript
@@filename(main)
import { HttpStatus } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';
import { AppService } from './app.service';

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const appService = appContext.get(AppService);

  return {
    body: appService.getHello(),
    statusCode: HttpStatus.OK,
  };
};
```

> 信息提示：请注意`NestFactory.createApplicationContext`不会用增强器（守卫、拦截器等）包装控制器方法。为此，您必须使用`NestFactory.create`方法。

您还可以将`event`对象传递给`EventsService`提供者，该提供者可以处理它并返回相应的值（取决于输入值和您的业务逻辑）。

```typescript
export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const eventsService = appContext.get(EventsService);
  return eventsService.process(event);
};
```