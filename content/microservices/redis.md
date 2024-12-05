### Redis

Redis 传输器实现了发布/订阅消息范式，并利用了 Redis 的 [Pub/Sub](https://redis.io/topics/pubsub) 特性。发布的消息被分类到频道中，发布者不知道最终会有哪些订阅者（如果有的话）接收到消息。每个微服务可以订阅任意数量的频道。此外，可以同时订阅多个频道。通过频道交换的消息是**即发即弃**的，这意味着如果一个消息被发布而没有订阅者对此感兴趣，消息将被移除且无法恢复。因此，你不能保证消息或事件至少会被一个服务处理。单个消息可以被多个订阅者订阅（和接收）。

<figure><img class="illustrative-image" src="/assets/Redis_1.png" /></figure>

#### 安装

要开始构建基于 Redis 的微服务，首先安装所需的包：

```bash
$ npm i --save ioredis
```

#### 概览

要使用 Redis 传输器，将以下选项对象传递给 `createMicroservice()` 方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
```

> 信息提示 **提示** `Transport` 枚举是从 `@nestjs/microservices` 包导入的。

#### 选项

`options` 属性是特定于所选传输器的。Redis 传输器暴露了以下描述的属性。

<table>
  <tr>
    <td><code>host</code></td>
    <td>连接 URL</td>
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
    <td>消息重试尝试之间的延迟（毫秒）（默认：<code>0</code>）</td>
  </tr>
  <tr>
    <td><code>wildcards</code></td>
    <td>启用 Redis 通配符订阅，指示传输器在底层使用 <code>psubscribe</code>/<code>pmessage</code>。（默认：<code>false</code>）</td>
  </tr>
</table>

官方 [ioredis](https://redis.github.io/ioredis/index.html#RedisOptions) 客户端支持的所有属性也由这个传输器支持。

#### 客户端

像其他微服务传输器一样，你有[多种选项](https://docs.nestjs.com/microservices/basics#client)来创建 Redis `ClientProxy` 实例。

创建客户端实例的一种方法是使用 `ClientsModule`。要使用 `ClientsModule` 创建客户端实例，导入它并使用 `register()` 方法传递一个选项对象，该对象具有与上述 `createMicroservice()` 方法中显示的相同属性，以及一个 `name` 属性，用作注入令牌。更多关于 `ClientsModule` 的信息[点击这里](https://docs.nestjs.com/microservices/basics#client)。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.REDIS,
        options: {
          host: 'localhost',
          port: 6379,
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他选项创建客户端（无论是 `ClientProxyFactory` 还是 `@Client()`）。你可以[点击这里](https://docs.nestjs.com/microservices/basics#client)了解更多信息。

#### 上下文

在更复杂的情况下，你可能想要访问更多关于传入请求的信息。当使用 Redis 传输器时，你可以访问 `RedisContext` 对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RedisContext) {
  console.log(`Channel: ${context.getChannel()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Channel: ${context.getChannel()}`);
}
```

> 信息提示 **提示** `@Payload()`, `@Ctx()` 和 `RedisContext` 是从 `@nestjs/microservices` 包导入的。