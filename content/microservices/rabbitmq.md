### RabbitMQ

[RabbitMQ](https://www.rabbitmq.com/) 是一个开源且轻量级的消息代理，支持多种消息协议。它可以在分布式和联合配置中部署，以满足高规模、高可用性的需求。此外，它还是全球部署最广泛的消息代理，从小型企业到大型企业都在使用。

#### 安装

要开始构建基于 RabbitMQ 的微服务，首先安装所需的包：

```bash
$ npm i --save amqplib amqp-connection-manager
```

#### 概览

要使用 RabbitMQ 传输器，请将以下选项对象传递给 `createMicroservice()` 方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
```

> 信息 **提示** `Transport` 枚举是从 `@nestjs/microservices` 包中导入的。

#### 选项

`options` 属性是特定于所选传输器的。`RabbitMQ` 传输器暴露了以下属性。

<table>
  <tr>
    <td><code>urls</code></td>
    <td>连接 URL</td>
  </tr>
  <tr>
    <td><code>queue</code></td>
    <td>服务器将监听的队列名称</td>
  </tr>
  <tr>
    <td><code>prefetchCount</code></td>
    <td>设置通道的预取计数</td>
  </tr>
  <tr>
    <td><code>isGlobalPrefetchCount</code></td>
    <td>启用每个通道的预取</td>
  </tr>
  <tr>
    <td><code>noAck</code></td>
    <td>如果为 <code>false</code>，则启用手动确认模式</td>
  </tr>
  <tr>
    <td><code>consumerTag</code></td>
    <td>消费者标签标识符（更多信息请 <a href="https://amqp-node.github.io/amqplib/channel_api.html#channel_consume" rel="nofollow" target="_blank">点击这里</a>）</td>
  </tr>
  <tr>
    <td><code>queueOptions</code></td>
    <td>额外的队列选项（更多信息请 <a href="https://amqp-node.github.io/amqplib/channel_api.html#channel_assertQueue" rel="nofollow" target="_blank">点击这里</a>）</td>
  </tr>
  <tr>
    <td><code>socketOptions</code></td>
    <td>额外的套接字选项（更多信息请 <a href="https://amqp-node.github.io/amqplib/channel_api.html#connect" rel="nofollow" target="_blank">点击这里</a>）</td>
  </tr>
  <tr>
    <td><code>headers</code></td>
    <td>随每条消息一起发送的头部</td>
  </tr>
</table>

#### 客户端

像其他微服务传输器一样，您有 <a href="https://docs.nestjs.com/microservices/basics#client">几种选项</a> 来创建 RabbitMQ `ClientProxy` 实例。

使用 `ClientsModule` 创建客户端实例的一种方法是导入它并使用 `register()` 方法传递一个选项对象，该对象具有与 `createMicroservice()` 方法中显示的相同属性，以及一个 `name` 属性，用作注入令牌。更多关于 `ClientsModule` 的信息，请 <a href="https://docs.nestjs.com/microservices/basics#client">点击这里</a>。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.RMQ,
        options: {
          urls: ['amqp://localhost:5672'],
          queue: 'cats_queue',
          queueOptions: {
            durable: false
          },
        },
      },
    ]),
  ]
  ...
})
```

也可以使用其他选项创建客户端（无论是 `ClientProxyFactory` 还是 `@Client()`）。您可以在 <a href="https://docs.nestjs.com/microservices/basics#client">这里</a> 阅读有关它们的更多信息。

#### 上下文

在更复杂的场景中，您可能想要访问有关传入请求的更多信息。当使用 RabbitMQ 传输器时，您可以访问 `RmqContext` 对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(`Pattern: ${context.getPattern()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Pattern: ${context.getPattern()}`);
}
```

> 信息 **提示** `@Payload()`, `@Ctx()` 和 `RmqContext` 是从 `@nestjs/microservices` 包中导入的。

要访问原始的 RabbitMQ 消息（带有 `properties`, `fields` 和 `content`），请使用 `RmqContext` 对象的 `getMessage()` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getMessage());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getMessage());
}
```

要检索 RabbitMQ [通道](https://www.rabbitmq.com/channels.html) 的引用，请使用 `RmqContext` 对象的 `getChannelRef` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getChannelRef());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getChannelRef());
}
```

#### 消息确认

为确保消息永远不会丢失，RabbitMQ 支持 [消息确认](https://www.rabbitmq.com/confirms.html)。确认是由消费者发送回 RabbitMQ 的，以告知特定消息已被接收、处理，并且 RabbitMQ 可以自由删除它。如果消费者死亡（其通道关闭、连接关闭或 TCP 连接丢失）而没有发送确认，RabbitMQ 将理解消息没有被完全处理，并将重新入队。

要启用手动确认模式，请将 `noAck` 属性设置为 `false`：

```typescript
options: {
  urls: ['amqp://localhost:5672'],
  queue: 'cats_queue',
  noAck: false,
  queueOptions: {
    durable: false
  },
},
```

当手动消费者确认打开时，我们必须从工作器发送适当的确认，以表示我们已经完成了任务。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
```

#### 记录构建器

要配置消息选项，您可以使用 `RmqRecordBuilder` 类（注意：这对于基于事件的流程也是可行的）。例如，要设置 `headers` 和 `priority` 属性，请使用 `setOptions` 方法，如下所示：

```typescript
const message = ':cat:';
const record = new RmqRecordBuilder(message)
  .setOptions({
    headers: {
      ['x-version']: '1.0.0',
    },
    priority: 3,
  })
  .build();

this.client.send('replace-emoji', record).subscribe(...);
```

> 信息 **提示** `RmqRecordBuilder` 类是从 `@nestjs/microservices` 包中导出的。

您也可以在服务器端通过访问 `RmqContext` 来读取这些值，如下所示：

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: RmqContext): string {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```