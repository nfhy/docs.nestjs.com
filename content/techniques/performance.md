### 性能（Fastify）

默认情况下，Nest 使用的是 [Express](https://expressjs.com/) 框架。正如前面提到的，Nest 还提供了与其他库的兼容性，例如 [Fastify](https://github.com/fastify/fastify)。Nest 通过实现框架适配器来实现框架独立性，其主要功能是将中间件和处理器代理到适当的库特定的实现。

> 信息 **提示** 请注意，为了实现框架适配器，目标库必须提供与 Express 中类似的请求/响应管道处理。

[Fastify](https://github.com/fastify/fastify) 为 Nest 提供了一个不错的替代框架，因为它以类似 Express 的方式解决了设计问题。然而，Fastify 比 Express **更快**，在基准测试结果上几乎快了两倍。一个公平的问题是，为什么 Nest 使用 Express 作为默认的 HTTP 提供者？原因是 Express 被广泛使用，知名度高，并且有大量的兼容中间件，Nest 用户可以立即使用。

但由于 Nest 提供了框架独立性，你可以轻松地在它们之间迁移。当你非常重视快速性能时，Fastify 可能是一个更好的选择。要使用 Fastify，只需选择内置的 `FastifyAdapter`，如下所示。

#### 安装

首先，我们需要安装所需的包：

```bash
$ npm i --save @nestjs/platform-fastify
```

#### 适配器

一旦安装了 Fastify 平台，我们就可以使用方法 `FastifyAdapter`。

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter()
  );
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

默认情况下，Fastify 仅监听 `localhost 127.0.0.1` 接口（[了解更多](https://www.fastify.io/docs/latest/Guides/Getting-Started/#your-first-server)）。如果你想要在其他主机上接受连接，你应该在 `listen()` 调用中指定 `'0.0.0.0'`：

```typescript
async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  await app.listen(3000, '0.0.0.0');
}
```

#### 平台特定包

请注意，当你使用 `FastifyAdapter` 时，Nest 使用 Fastify 作为 **HTTP 提供者**。这意味着每个依赖于 Express 的食谱可能不再有效。你应该使用 Fastify 等效的包。

#### 重定向响应

Fastify 处理重定向响应的方式与 Express 略有不同。要正确使用 Fastify 进行重定向，请返回状态码和 URL，如下所示：

```typescript
@Get()
index(@Res() res) {
  res.status(302).redirect('/login');
}
```

#### Fastify 选项

你可以通过 `FastifyAdapter` 构造函数将选项传递给 Fastify 构造函数。例如：

```typescript
new FastifyAdapter({ logger: true });
```

#### 中间件

中间件函数检索原始的 `req` 和 `res` 对象，而不是 Fastify 的包装器。这就是 `middie` 包的工作方式（在底层使用），`fastify` - 查看这个 [页面](https://www.fastify.io/docs/latest/Reference/Middleware/) 了解更多信息，

```typescript
@@filename(logger.middleware)
import { Injectable, NestMiddleware } from '@nestjs/common';
import { FastifyRequest, FastifyReply } from 'fastify';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: FastifyRequest['raw'], res: FastifyReply['raw'], next: () => void) {
    console.log('Request...');
    next();
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerMiddleware {
  use(req, res, next) {
    console.log('Request...');
    next();
  }
}
```

#### 路由配置

你可以使用 Fastify 的 [路由配置](https://fastify.dev/docs/latest/Reference/Routes/#config) 功能与 `@RouteConfig()` 装饰器。

```typescript
@RouteConfig({ output: 'hello world' })
@Get()
index(@Req() req) {
  return req.routeConfig.output;
}
```

#### 路由约束

从 v10.3.0 开始，`@nestjs/platform-fastify` 支持 Fastify 的 [路由约束](https://fastify.dev/docs/latest/Reference/Routes/#constraints) 功能与 `@RouteConstraints` 装饰器。

```typescript
@RouteConstraints({ version: '1.2.x' })
newFeature() {
  return 'This works only for version >= 1.2.x';
}
```

> 信息 **提示** `@RouteConfig()` 和 `@RouteConstraints` 是从 `@nestjs/platform-fastify` 导入的。

#### 示例

一个工作示例可以在 [这里](https://github.com/nestjs/nest/tree/master/sample/10-fastify) 找到。