### NATS

[NATS](https://nats.io) 是一个简单、安全且高性能的开源消息系统，适用于云原生应用、IoT消息传递和微服务架构。NATS服务器是用Go语言编写的，但与服务器交互的客户端库适用于数十种主要编程语言。NATS支持**至多一次**和**至少一次**的消息传递。它可以在任何地方运行，从大型服务器和云实例，到边缘网关甚至物联网设备。

#### 安装

要开始构建基于NATS的微服务，首先安装所需的包：

```bash
$ npm i --save nats
```

#### 概览

要使用NATS传输器，请将以下选项对象传递给 `createMicroservice()` 方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
```

> 信息 **提示** `Transport` 枚举是从 `@nestjs/microservices` 包导入的。

#### 选项

`options` 对象特定于所选的传输器。`NATS` 传输器公开了[这里](https://github.com/nats-io/node-nats#connection-options)描述的属性。此外，还有一个 `queue` 属性，允许你指定服务器应该订阅的队列名称（留空 `undefined` 以忽略此设置）。更多关于NATS队列组的信息，请[点击这里](https://docs.nestjs.com/microservices/nats#queue-groups)。

#### 客户端

像其他微服务传输器一样，你有[多种选项](https://docs.nestjs.com/microservices/basics#client)来创建NATS `ClientProxy` 实例。

使用 `ClientsModule` 创建客户端实例的一种方法是导入它，并使用 `register()` 方法传递一个选项对象，该对象具有与上面 `createMicroservice()` 方法中显示的相同属性，以及一个 `name` 属性，用作注入令牌。更多关于 `ClientsModule` 的信息，请[点击这里](https://docs.nestjs.com/microservices/basics#client)。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.NATS,
        options: {
          servers: ['nats://localhost:4222'],
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他选项创建客户端（无论是 `ClientProxyFactory` 还是 `@Client()`）。你可以[点击这里](https://docs.nestjs.com/microservices/basics#client)了解更多信息。

#### 请求-响应

对于**请求-响应**消息风格（[了解更多](https://docs.nestjs.com/microservices/basics#request-response)），NATS传输器不使用NATS内置的[请求-回复](https://docs.nats.io/nats-concepts/reqreply)机制。相反，使用 `publish()` 方法在给定的主题上发布“请求”，并使用一个唯一的回复主题名称，响应者监听该主题并将响应发送到回复主题。回复主题动态地指向请求者，无论双方的位置如何。

#### 基于事件

对于**基于事件**的消息风格（[了解更多](https://docs.nestjs.com/microservices/basics#event-based)），NATS传输器使用NATS内置的[发布-订阅](https://docs.nats.io/nats-concepts/pubsub)机制。发布者在主题上发送消息，任何活跃的订阅者监听该主题都会接收到消息。订阅者还可以注册对通配符主题的兴趣，这些主题有点像正则表达式。这种一对多模式有时被称为广播。

#### 队列组

NATS提供了一个内置的负载均衡功能，称为[分布式队列](https://docs.nats.io/nats-concepts/queue)。要创建队列订阅，请使用 `queue` 属性如下：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
    queue: 'cats_queue',
  },
});
```

#### 上下文

在更复杂的场景中，你可能想要访问更多关于传入请求的信息。当使用NATS传输器时，你可以访问 `NatsContext` 对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Subject: ${context.getSubject()}`);
}
```

> 信息 **提示** `@Payload()`, `@Ctx()` 和 `NatsContext` 是从 `@nestjs/microservices` 包导入的。

#### 通配符

订阅可以是明确的主体，也可以包含通配符。

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

#### 记录构建器

要配置消息选项，你可以使用 `NatsRecordBuilder` 类（注意：这对于基于事件的流程也是可行的）。例如，要添加 `x-version` 头部，使用 `setHeaders` 方法，如下所示：

```typescript
import * as nats from 'nats';

// 代码中的某个位置
const headers = nats.headers();
headers.set('x-version', '1.0.0');

const record = new NatsRecordBuilder(':cat:').setHeaders(headers).build();
this.client.send('replace-emoji', record).subscribe(...);
```

> 信息 **提示** `NatsRecordBuilder` 类是从 `@nestjs/microservices` 包导出的。

你也可以通过访问 `NatsContext` 在服务器端读取这些头部，如下所示：

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: NatsContext): string {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

在某些情况下，你可能想要为多个请求配置头部，你可以将这些作为选项传递给 `ClientProxyFactory`：

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.NATS,
          options: {
            servers: ['nats://localhost:4222'],
            headers: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```