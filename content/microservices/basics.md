### 概览

除了传统的（有时称为单体的）应用程序架构，Nest 本身支持微服务架构风格开发。本文档中讨论的大多数概念，如依赖注入、装饰器、异常过滤器、管道、守卫和拦截器，同样适用于微服务。在尽可能的情况下，Nest 抽象了每个传输层的实现细节，以便相同的组件可以在基于 HTTP 的平台、WebSocket 和微服务中运行。本节涵盖了 Nest 中特定于微服务的方面。

在 Nest 中，微服务基本上是使用与 HTTP 不同的**传输**层的应用程序。

<figure><img class="illustrative-image" src="/assets/Microservices_1.png" /></figure>

Nest 支持几种内置的传输层实现，称为**传输器**，它们负责在不同的微服务实例之间传输消息。大多数传输器原生支持**请求-响应**和**基于事件**的消息风格。Nest 在两种请求-响应和基于事件的消息传递背后抽象了每个传输器的实现细节。这使得从一个传输层切换到另一个传输层变得容易——例如，利用特定传输层的特定可靠性或性能特性——而不会影响您的应用程序代码。

#### 安装

要开始构建微服务，首先安装所需的包：

```bash
$ npm i --save @nestjs/microservices
```

#### 快速开始

要实例化一个微服务，使用 `NestFactory` 类的 `createMicroservice()` 方法：

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
    },
  );
  await app.listen();
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.TCP,
  });
  await app.listen();
}
bootstrap();
```

> 信息 **提示** 微服务默认使用 **TCP** 传输层。

`createMicroservice()` 方法的第二个参数是一个 `options` 对象。这个对象可能包含两个成员：

<table>
  <tr>
    <td><code>transport</code></td>
    <td>指定传输器（例如，<code>Transport.NATS</code>）</td>
  </tr>
  <tr>
    <td><code>options</code></td>
    <td>一个特定于传输器的选项对象，它决定了传输器的行为</td>
  </tr>
</table>

`options` 对象特定于所选的传输器。`TCP` 传输器暴露了以下属性。对于其他传输器（例如 Redis、MQTT 等），请参见相关章节以了解可用选项的描述。

<table>
  <tr>
    <td><code>host</code></td>
    <td>连接主机名</td>
  </tr>
  <tr>
    <td><code>port</code></td>
    <td>连接端口</td>
  </tr>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>消息重试次数（默认：<code>0</code>）</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>消息重试尝试之间的延迟（ms）（默认：<code>0</code>）</td>
  </tr>
  <tr>
    <td><code>serializer</code></td>
    <td>自定义<a href="https://github.com/nestjs/nest/blob/master/packages/microservices/interfaces/serializer.interface.ts" target="_blank">序列化器</a>用于传出消息</td>
  </tr>
  <tr>
    <td><code>deserializer</code></td>
    <td>自定义<a href="https://github.com/nestjs/nest/blob/master/packages/microservices/interfaces/deserializer.interface.ts" target="_blank">反序列化器</a>用于传入消息</td>
  </tr>
  <tr>
    <td><code>socketClass</code></td>
    <td>自定义 Socket 扩展 <code>TcpSocket</code>（默认：<code>JsonSocket</code>）</td>
  </tr>
  <tr>
    <td><code>tlsOptions</code></td>
    <td>配置 tls 协议的选项</td>
  </tr>
</table>

#### 模式

微服务通过**模式**识别消息和事件。模式是一个纯值，例如，一个字面量对象或一个字符串。模式会自动序列化并随消息的数据部分一起通过网络发送。通过这种方式，消息发送者和消费者可以协调哪些请求由哪些处理器消费。

#### 请求-响应

请求-响应消息风格在需要在各种外部服务之间交换消息时非常有用。有了这种范式，您可以确定服务实际上已经接收到了消息（无需手动实现消息确认协议）。然而，请求-响应范式并不总是最佳选择。例如，使用基于日志的持久性的流传输器，如 [Kafka](https://docs.confluent.io/3.0.0/streams/) 或 [NATS streaming](https://github.com/nats-io/node-nats-streaming)，它们优化了解决一系列不同的问题，更符合事件消息范式（有关详细信息，请参阅下面的[基于事件的消息传递](https://docs.nestjs.com/microservices/basics#event-based)）。

要启用请求-响应消息类型，Nest 创建了两个逻辑通道——一个负责传输数据，另一个等待传入响应。对于某些底层传输，如 [NATS](https://nats.io/)，这种双通道支持是开箱即用的。对于其他传输，Nest 通过手动创建单独的通道来补偿。这可能会有开销，因此如果您不需要请求-响应消息风格，您应该考虑使用基于事件的方法。

要基于请求-响应范式创建消息处理器，请使用 `@MessagePattern()` 装饰器，该装饰器从 `@nestjs/microservices` 包导入。此装饰器仅应在 [控制器](https://docs.nestjs.com/controllers) 类中使用，因为它们是您应用程序的入口点。在提供者中使用它们不会有任何效果，因为它们会被 Nest 运行时忽略。

```typescript
@@filename(math.controller)
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data: number[]): number {
    return (data || []).reduce((a, b) => a + b);
  }
}
@@switch
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data) {
    return (data || []).reduce((a, b) => a + b);
  }
}
```

在上述代码中，`accumulate()` **消息处理器**侦听满足 `{{ '{' }} cmd: 'sum' {{ '}' }}` 消息模式的消息。消息处理器接受一个参数，即从客户端传递的 `data`。在这种情况下，数据是一个数字数组，需要累积。

#### 异步响应

消息处理器能够同步或**异步**响应。因此，支持 `async` 方法。

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
async accumulate(data: number[]): Promise<number> {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@MessagePattern({ cmd: 'sum' })
async accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```

消息处理器还能够返回一个 `Observable`，在这种情况下，结果值将被发出，直到流完成。

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
@@switch
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
```

在上面的例子中，消息处理器将响应 **3次**（数组中的每个项目）。

#### 基于事件

虽然请求-响应方法适用于服务间的消息交换，但当您的消息风格基于事件时，它就不那么适合了——当您只想发布**事件**而不等待响应时。在这种情况下，您不希望请求-响应所需的开销来维护两个通道。

假设您只是想通知另一个服务，这部分系统发生了某个条件。这是基于事件的消息风格的典型用例。

要创建事件处理器，我们使用 `@EventPattern()` 装饰器，该装饰器从 `@nestjs/microservices` 包导入。

```typescript
@@filename()
@EventPattern('user_created')
async handleUserCreated(data: Record<string, unknown>) {
  // 业务逻辑
}
@@switch
@EventPattern('user_created')
async handleUserCreated(data) {
  // 业务逻辑
}
```

> 信息 **提示** 您可以为 **单个** 事件模式注册多个事件处理器，所有这些处理器将自动并行触发。

`handleUserCreated()` **事件处理器**侦听 `'user_created'` 事件。事件处理器接受一个参数，即从客户端传递的 `data`（在这种情况下，是一个事件负载，已通过网络发送）。

#### 装饰器

在更复杂的情况下，您可能想要访问有关传入请求的更多信息。例如，在 NATS 带有通配符订阅的情况下，您可能想要获取生产者发送消息的原始主题。同样，在 Kafka 中，您可能想要访问消息头。为了实现这一点，您可以使用以下内置装饰器：

```typescript
@@filename()
@MessagePattern('time.us.*')
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*')
getDate(data, context) {
  console.log(`Subject: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
```

> 信息 **提示** `@Payload()`, `@Ctx()` 和 `NatsContext` 从 `@nestjs/microservices` 导入。

> 信息 **提示** 您还可以向 `@Payload()` 装饰器传递一个属性键，以从传入的负载对象中提取特定属性，例如 `@Payload('id')`。

#### 客户端

Nest 应用程序客户端可以使用 `ClientProxy` 类与 Nest 微服务交换消息或发布事件。这个类定义了几种方法，如 `send()`（用于请求-响应消息传递）和 `emit()`（用于事件驱动消息传递），让您可以与远程微服务通信。可以通过以下方式之一获得这个类的实例。

一种技术是导入 `ClientsModule`，它公开了静态 `register()` 方法。此方法接受一个参数，该参数是一个对象数组，表示微服务传输器。每个这样的对象都有一个 `name` 属性，一个可选的 `transport` 属性（默认为 `Transport.TCP`），以及一个可选的特定于传输器的 `options` 属性。

`name` 属性作为 **注入令牌**，可以用来在需要的地方注入 `ClientProxy` 的实例。`name` 属性的值，作为注入令牌，可以是一个任意字符串或 JavaScript 符号，如[这里](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens)所述。

`options` 属性是一个包含我们在 `createMicroservice()` 方法中看到相同属性的对象。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      { name: 'MATH_SERVICE', transport: Transport.TCP },
    ]),
  ],
})
```

或者，如果您需要传递配置或执行任何其他异步过程，可以使用 `registerAsync()` 方法。

```typescript
@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        imports: [ConfigModule],
        name: 'MATH_SERVICE',
        useFactory: async (configService: ConfigService) => ({
          transport: Transport.TCP,
          options: {
            url: configService.get('URL'),
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
})
```

一旦模块被导入，您可以使用 `@Inject()` 装饰器注入配置为 `'MATH_SERVICE'` 传输器指定选项的 `ClientProxy` 实例。

```typescript
constructor(
  @Inject('MATH_SERVICE') private client: ClientProxy,
) {}
```

> 信息 **提示** `ClientsModule` 和 `ClientProxy` 类从 `@nestjs/microservices` 包导入。

有时我们可能需要从另一个服务（比如 `ConfigService`）获取传输器配置，而不是在客户端应用程序中硬编码它。为此，我们可以注册一个 [自定义提供者](/fundamentals/custom-providers) 使用 `ClientProxyFactory` 类。这个类有一个静态 `create()` 方法，它接受一个传输器选项对象，并返回一个定制的 `ClientProxy` 实例。

```typescript
@Module({
  providers: [
    {
      provide: 'MATH_SERVICE',
      useFactory: (configService: ConfigService) => {
        const mathSvcOptions = configService.getMathSvcOptions();
        return ClientProxyFactory.create(mathSvcOptions);
      },
      inject: [ConfigService],
    }
  ]
  ...
})
```

> 信息 **提示** `ClientProxyFactory` 从 `@nestjs/microservices` 包导入。

另一个选项是使用 `@Client()` 属性装饰器。

```typescript
@Client({ transport: Transport.TCP })
client: ClientProxy;
```

> 信息 **提示** `@Client()` 装饰器从 `@nestjs/microservices` 包导入。

使用 `@Client()` 装饰器不是首选技术，因为它更难测试，更难共享客户端实例。

`ClientProxy` 是 **懒加载** 的。它不会立即启动连接。相反，它将在第一次微服务调用之前建立连接，然后在整个后续调用中重用。然而，如果您希望将应用程序引导过程延迟到建立连接之后，您可以在 `OnApplicationBootstrap` 生命周期钩子内手动使用 `ClientProxy` 对象的 `connect()` 方法启动连接。

```typescript
@@filename()
async onApplicationBootstrap() {
  await this.client.connect();
}
```

如果无法创建连接，`connect()` 方法将用相应的错误对象拒绝。

#### 发送消息

`ClientProxy` 公开了一个 `send()` 方法。这个方法旨在调用微服务并返回一个 `Observable` 作为其响应。因此，我们可以轻松地订阅到发出的值。

```typescript
@@filename()
accumulate(): Observable<number> {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send<number>(pattern, payload);
}
@@switch
accumulate() {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send(pattern, payload);
}
```

`send()` 方法接受两个参数，`pattern` 和 `payload`。`pattern` 应该与 `@MessagePattern()` 装饰器中定义的一个匹配。`payload` 是我们想要传输到远程微服务的消息。这个方法返回一个 **冷 `Observable`**，这意味着您必须显式订阅它，然后消息才会被发送。

#### 发布事件

要发送事件，请使用 `ClientProxy` 对象的 `emit()` 方法。这个方法将事件发布到消息代理。

```typescript
@@filename()
async publish() {
  this.client.emit<number>('user_created', new UserCreatedEvent());
}
@@switch
async publish() {
  this.client.emit('user_created', new UserCreatedEvent());
}
```

`emit()` 方法接受两个参数，`pattern` 和 `payload`。`pattern` 应该与 `@EventPattern()` 装饰器中定义的一个匹配。`payload` 是我们想要传输到远程微服务的事件负载。这个方法返回一个 **热 `Observable`**（与 `send()` 返回的冷 `Observable` 不同），这意味着无论您是否显式订阅该 observable，代理都会立即尝试传递事件。

#### 作用域

对于来自不同编程语言背景的人来说，了解到在 Nest 中，几乎所有东西都是跨传入请求共享的，这可能是出乎意料的。我们有一个数据库连接池，具有全局状态的单例服务等。请记住，Node.js 不遵循每个请求由单独线程处理的请求/响应多线程无状态模型。因此，使用单例实例对我们的应用程序来说是完全 **安全** 的。

然而，有一些边缘情况，当请求基础的处理器生命周期可能是期望的行为时，例如 GraphQL 应用程序中的每个请求的缓存，请求跟踪或多租户。了解如何控制作用域 [here](/fundamentals/injection-scopes)。

请求作用域的处理器和提供者可以使用 `@Inject()` 装饰器结合 `CONTEXT` 令牌注入 `RequestContext`：

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT, RequestContext } from '@nestjs/microservices';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private ctx: RequestContext) {}
}
```

这提供了对 `RequestContext` 对象的访问，它有两个属性：

```typescript
export interface RequestContext<T = any> {
  pattern: string | Record<string, any>;
  data: T;
}
```

`data` 属性是由消息生产者发送的消息负载。`pattern` 属性是用于识别适当处理器处理传入消息的模式。

#### 处理超时

在分布式系统中，有时微服务可能会宕机或不可用。为了避免无限期等待，您可以使用超时。在与其他服务通信时，超时是一个非常有用的模式。要将超时应用于您的微服务调用，您可以使用 [RxJS](https://rxjs.dev) `timeout` 操作符。如果微服务在一定时间内没有响应请求，将抛出异常，您可以捕获并适当处理。

要解决这个问题，您必须使用 [`rxjs`](https://github.com/ReactiveX/rxjs) 包。只需在管道中使用 `timeout` 操作符：

```typescript
@@filename()
this.client
      .send<TResult, TInput>(pattern, data)
      .pipe(timeout(5000));
@@switch
this.client
      .send(pattern, data)
      .pipe(timeout(5000));
```

> 信息 **提示** `timeout` 操作符从 `rxjs/operators` 包导入。

5秒后，如果微服务没有响应，它将抛出一个错误。

#### TLS 支持

当在私有网络之外通信时，加密流量以确保安全是非常重要的。在 NestJS 中，这可以通过使用 Node 的内置 [TLS](https://nodejs.org/api/tls.html) 模块在 TCP 上实现 TLS 来完成。Nest 在其 TCP 传输中提供了 TLS 的内置支持，允许我们在微服务或客户端之间加密通信。

要为 TCP 服务器启用 TLS，您需要一个私钥和一个 PEM 格式的证书。这些通过设置 `tlsOptions` 并指定密钥和证书文件添加到服务器的选项中，如下所示：

```typescript
import * as fs from 'fs';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const key = fs.readFileSync('<pathToKeyFile>', 'utf8').toString();
  const cert = fs.readFileSync('<pathToCertFile>', 'utf8').toString();

  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        tlsOptions: {
          key,
          cert,
        },
      },
    },
  );

  await app.listen();
}
bootstrap();
```

对于客户端要安全地通过 TLS 通信，我们也定义 `tlsOptions` 对象，但这次是带有 CA 证书。这是签署服务器证书的权威的证书。这确保客户端信任服务器的证书，并可以建立安全连接。

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.TCP,
        options: {
          tlsOptions: {
            ca: [fs.readFileSync('<pathToCaFile>', 'utf-8').toString()],
          },
        },
      },
    ]),
  ],
})
export class AppModule {}
```

您也可以传递一个 CA 数组，如果您的设置涉及多个受信任的权威。

一旦一切设置完成，您可以像往常一样使用 `@Inject()` 装饰器注入 `ClientProxy` 在您的服务中使用客户端。这确保了您的 NestJS 微服务之间的通信是加密的，Node 的 `TLS` 模块处理加密细节。

更多信息，请参考 Node 的 [TLS 文档](https://nodejs.org/api/tls.html)。