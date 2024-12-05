获取原始请求体的最常见用例之一是执行Webhook签名验证。通常，为了执行Webhook签名验证，需要未序列化的请求体来计算HMAC哈希。

> 警告 **警告** 此功能只能在启用内置全局body解析器中间件时使用，即在创建应用时不能传递 `bodyParser: false`。

#### 在Express中使用

首先，在创建Nest Express应用程序时启用此选项：

```typescript
import { NestFactory } from '@nestjs/core';
import type { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

// 在 "bootstrap" 函数中
const app = await NestFactory.create<NestExpressApplication>(AppModule, {
  rawBody: true,
});
await app.listen(process.env.PORT ?? 3000);
```

在控制器中访问原始请求体，提供了一个方便的接口 `RawBodyRequest` 来在请求上暴露一个 `rawBody` 字段：使用接口 `RawBodyRequest` 类型：

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<Request>) {
    const raw = req.rawBody; // 返回一个 `Buffer`。
  }
}
```

#### 注册不同的解析器

默认情况下，只注册了 `json` 和 `urlencoded` 解析器。如果你想要在运行时注册不同的解析器，你需要显式这样做。

例如，要注册一个 `text` 解析器，你可以使用以下代码：

```typescript
app.useBodyParser('text');
```

> 警告 **警告** 确保你在 `NestFactory.create` 调用中提供了正确的应用程序类型。对于Express应用程序，正确的类型是 `NestExpressApplication`。否则 `.useBodyParser` 方法将找不到。

#### 体解析器大小限制

如果你的应用程序需要解析大于Express默认的 `100kb` 的体，使用以下代码：

```typescript
app.useBodyParser('json', { limit: '10mb' });
```

`.useBodyParser` 方法将尊重传递给应用程序选项的 `rawBody` 选项。

#### 在Fastify中使用

首先，在创建Nest Fastify应用程序时启用此选项：

```typescript
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

// 在 "bootstrap" 函数中
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
  {
    rawBody: true,
  },
);
await app.listen(process.env.PORT ?? 3000);
```

在控制器中访问原始请求体，提供了一个方便的接口 `RawBodyRequest` 来在请求上暴露一个 `rawBody` 字段：使用接口 `RawBodyRequest` 类型：

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { FastifyRequest } from 'fastify';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<FastifyRequest>) {
    const raw = req.rawBody; // 返回一个 `Buffer`。
  }
}
```

#### 注册不同的解析器

默认情况下，只注册了 `application/json` 和 `application/x-www-form-urlencoded` 解析器。如果你想要在运行时注册不同的解析器，你需要显式这样做。

例如，要注册一个 `text/plain` 解析器，你可以使用以下代码：

```typescript
app.useBodyParser('text/plain');
```

> 警告 **警告** 确保你在 `NestFactory.create` 调用中提供了正确的应用程序类型。对于Fastify应用程序，正确的类型是 `NestFastifyApplication`。否则 `.useBodyParser` 方法将找不到。

#### 体解析器大小限制

如果你的应用程序需要解析大于Fastify默认的1MiB的体，使用以下代码：

```typescript
const bodyLimit = 10_485_760; // 10MiB
app.useBodyParser('application/json', { bodyLimit });
```

`.useBodyParser` 方法将尊重传递给应用程序选项的 `rawBody` 选项。