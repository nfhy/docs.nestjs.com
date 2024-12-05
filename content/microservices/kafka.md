### Kafka

[Kafka](https://kafka.apache.org/) 是一个开源的分布式流处理平台，它具有以下三个关键功能：

- 发布和订阅记录流，类似于消息队列或企业消息系统。
- 以容错和持久化的方式存储记录流。
- 处理实时发生的记录流。

Kafka 项目旨在提供一个统一的、高吞吐量、低延迟的平台，用于处理实时数据流。它与 Apache Storm 和 Spark 集成得非常好，用于实时流数据分析。

#### 安装

要开始构建基于 Kafka 的微服务，首先安装所需的包：

```bash
$ npm i --save kafkajs
```

#### 概览

像其他 Nest 微服务传输层实现一样，你可以通过传递给 `createMicroservice()` 方法的选项对象中的 `transport` 属性来选择 Kafka 传输机制，以及一个可选的 `options` 属性，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
```

> 信息提示 **提示** `Transport` 枚举是从 `@nestjs/microservices` 包中导入的。

#### 选项

`options` 属性特定于所选的传输器。`Kafka` 传输器暴露了以下属性。

| 属性       | 描述                                                         |
|-----------|--------------------------------------------------------------|
| `client`  | 客户端配置选项（了解更多[这里](https://kafka.js.org/docs/configuration)） |
| `consumer`| 消费者配置选项（了解更多[这里](https://kafka.js.org/docs/consuming#a-name-options-a-options)） |
| `run`     | 运行配置选项（了解更多[这里](https://kafka.js.org/docs/consuming)） |
| `subscribe`| 订阅配置选项（了解更多[这里](https://kafka.js.org/docs/consuming#frombeginning)） |
| `producer`| 生产者配置选项（了解更多[这里](https://kafka.js.org/docs/producing#options)） |
| `send`    | 发送配置选项（了解更多[这里](https://kafka.js.org/docs/producing#options)） |
| `producerOnlyMode`| 功能标志，跳过消费者组注册，仅作为生产者（`boolean`） |
| `postfixId`| 更改客户端ID值的后缀（`string`） |

#### 客户端

与其它微服务传输器相比，Kafka 有一个小区别。我们不是使用 `ClientProxy` 类，而是使用 `ClientKafka` 类。

像其他微服务传输器一样，你有[多种选项](https://docs.nestjs.com/microservices/basics#client)来创建一个 `ClientKafka` 实例。

使用 `ClientsModule` 创建实例的一个方法是导入它，并使用 `register()` 方法传递一个选项对象，该对象具有上述 `createMicroservice()` 方法中显示的相同属性，以及一个 `name` 属性，用作注入令牌。了解更多关于 `ClientsModule` 的信息[这里](https://docs.nestjs.com/microservices/basics#client)。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'HERO_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: 'hero',
            brokers: ['localhost:9092'],
          },
          consumer: {
            groupId: 'hero-consumer'
          }
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他选项创建客户端（无论是 `ClientProxyFactory` 或 `@Client()`）。你可以[这里](https://docs.nestjs.com/microservices/basics#client)了解更多信息。

使用 `@Client()` 装饰器如下：

```typescript
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero',
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer'
    }
  }
})
client: ClientKafka;
```

#### 消息模式

Kafka 微服务消息模式为请求和回复通道使用两个主题。`ClientKafka#send()` 方法发送带有[回地址](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ReturnAddress.html)的消息，通过将[关联 ID](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html)、回复主题和回复分区与请求消息关联。这要求 `ClientKafka` 实例在发送消息之前订阅回复主题并至少分配给一个分区。

随后，你需要为每个运行的 Nest 应用程序至少有一个回复主题分区。例如，如果你运行了 4 个 Nest 应用程序，但回复主题只有 3 个分区，那么 1 个 Nest 应用程序在尝试发送消息时将出现错误。

当新的 `ClientKafka` 实例启动时，它们加入消费者组并订阅它们各自的主题。这个过程触发了消费者组的消费者分配的主题分区的重新平衡。

通常，主题分区是使用轮询分区器分配的，该分区器将主题分区分配给按消费者名称排序的消费者集合，这些名称在应用程序启动时随机设置。然而，当新消费者加入消费者组时，新消费者可以被放置在消费者集合中的任何位置。这造成了一个条件，即预先存在的消费者在重新平衡时被放置在新消费者之后，可能会被分配不同的分区。因此，被分配不同分区的消费者将丢失在重新平衡之前发送的请求的响应消息。

为了防止 `ClientKafka` 消费者丢失响应消息，使用了 Nest 特定的内置自定义分区器。这个自定义分区器将分区分配给按高分辨率时间戳（`process.hrtime()`）排序的消费者集合，这些时间戳在应用程序启动时设置。

#### 消息响应订阅

> 警告 **注意** 这一节仅与使用 [请求-响应](/microservices/basics#request-response) 消息样式（带有 `@MessagePattern` 装饰器和 `ClientKafka#send` 方法）相关。对于 [基于事件](/microservices/basics#event-based) 的通信（`@EventPattern` 装饰器和 `ClientKafka#emit` 方法），不需要订阅响应主题。

`ClientKafka` 类提供了 `subscribeToResponseOf()` 方法。`subscribeToResponseOf()` 方法接受请求的主题名称作为参数，并将派生的回复主题名称添加到回复主题集合中。在实现消息模式时需要此方法。

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
}
```

如果 `ClientKafka` 实例是异步创建的，`subscribeToResponseOf()` 方法必须在调用 `connect()` 方法之前调用。

```typescript
@@filename(heroes.controller)
async onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
  await this.client.connect();
}
```

#### 传入

Nest 以具有 `key`、`value` 和 `headers` 属性的对象形式接收传入的 Kafka 消息，这些属性的值类型为 `Buffer`。然后 Nest 通过将缓冲区转换为字符串来解析这些值。如果字符串是“对象类”的，Nest 尝试将字符串解析为 `JSON`。然后 `value` 被传递到其关联的处理程序。

#### 传出

Nest 在发布事件或发送消息时，通过序列化过程发送传出的 Kafka 消息。这发生在传递给 `ClientKafka` `emit()` 和 `send()` 方法的参数上，或者在 `@MessagePattern` 方法返回的值上。这种序列化通过使用 `JSON.stringify()` 或 `toString()` 原型方法将不是字符串或缓冲区的对象“字符串化”。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const dragonId = message.dragonId;
    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];
    return items;
  }
}
```

> 信息提示 `@Payload()` 是从 `@nestjs/microservices` 包中导入的。

传出的消息也可以通过传递带有 `key` 和 `value` 属性的对象来键控。键控消息对于满足[共分区要求](https://docs.confluent.io/current/ksql/docs/developer-guide/partition-data.html#co-partitioning-requirements)很重要。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        realm
      },
      key: heroId,
      value: items
    }
  }
}
```

此外，以这种格式传递的消息也可以包含在 `headers` 哈希属性中设置的自定义标头。标头哈希属性的值必须是 `string` 类型或 `Buffer` 类型。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        kafka_nestRealm: realm
      },
      key: heroId,
      value: items
    }
  }
}
```

#### 基于事件

虽然请求-响应方法非常适合在服务之间交换消息，但当你的消息风格是基于事件的（这反过来又非常适合 Kafka） - 当你只想发布事件而不需要等待响应时，它就不太适合了。在这种情况下，你不需要请求-响应所必需的开销来维护两个主题。

了解更多关于这方面的信息，请查看这两个部分：[概述：基于事件](/microservices/basics#event-based) 和 [概述：发布事件](/microservices/basics#publishing-events)。

#### 上下文

在更复杂的场景中，你可能想要访问更多关于传入请求的信息。当使用 Kafka 传输器时，你可以访问 `KafkaContext` 对象。

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

> 信息提示 `@Payload()`、`@Ctx()` 和 `KafkaContext` 是从 `@nestjs/microservices` 包中导入的。

要访问原始的 Kafka `IncomingMessage` 对象，使用 `KafkaContext` 对象的 `getMessage()` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
```

其中 `IncomingMessage` 满足以下接口：

```typescript
interface IncomingMessage {
  topic: string;
  partition: number;
  timestamp: string;
  size: number;
  attributes: number;
  offset: string;
  key: any;
  value: any;
  headers: Record<string, any>;
}
```

如果你的处理程序涉及每个接收到的消息的慢处理时间，你应该考虑使用 `heartbeat` 回调。要检索 `heartbeat` 函数，请使用 `KafkaContext` 的 `getHeartbeat()` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
async killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const heartbeat = context.getHeartbeat();

  // 执行一些慢处理
  await doWorkPart1();

  // 发送心跳以不超过 sessionTimeout
  await heartbeat();

  // 再次执行一些慢处理
  await doWorkPart2();
}
```

#### 命名约定

Kafka 微服务组件会在 `client.clientId` 和 `consumer.groupId` 选项上附加各自角色的描述，以防止 Nest 微服务客户端和服务器组件之间的冲突。默认情况下，`ClientKafka` 组件会在这两个选项上附加 `-client`，而 `ServerKafka` 组件会附加 `-server`。注意下面提供的值是如何以这种方式转换的（如注释所示）。

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-server
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-server
    },
  }
});
```

以及客户端：

```typescript
@@filename(heroes.controller)
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-client
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-client
    }
  }
})
client: ClientKafka;
```

> 信息提示 Kafka 客户端和消费者命名约定可以通过扩展 `ClientKafka` 和 `KafkaServer` 在你自己的自定义提供程序中重写构造函数来自定义。

由于 Kafka 微服务消息模式为请求和回复通道使用两个主题，因此应该从请求主题派生出回复模式。默认情况下，回复主题的名称是请求主题名称与附加的 `.reply` 的组合。

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.get'); // hero.get.reply
}
```

> 信息提示 Kafka 回复主题命名约定可以通过扩展 `ClientKafka` 在你自己的自定义提供程序中重写 `getResponsePatternName` 方法来自定义。

#### 可重试异常

与其他传输器类似，所有未处理的异常都会自动包装成 `RpcException` 并转换为“用户友好”的格式。然而，在某些边缘情况下，你可能想要绕过这种机制，让异常被 `kafkajs` 驱动程序消费。在处理消息时抛出异常会指示 `kafkajs` 将其**重试**（重新传递），这意味着尽管消息（或事件）处理程序被触发，但偏移量不会提交给 Kafka。

> 警告 **警告** 对于事件处理程序（基于事件的通信），所有未处理的异常默认都被视为**可重试异常**。

为此，你可以使用一个名为 `KafkaRetriableException` 的专用类，如下所示：

```typescript
throw new KafkaRetriableException('...');
```

> 信息提示 `KafkaRetriableException` 类是从 `@nestjs/microservices` 包中导出的。

#### 提交偏移量

提交偏移量在处理 Kafka 时至关重要。默认情况下，消息会在特定时间后自动提交。更多信息请访问 [KafkaJS 文档](https://kafka.js.org/docs/consuming#autocommit)。`KafkaContext` 提供了一种访问活动消费者以手动提交偏移量的方法。消费者是 KafkaJS 消费者，并且像 [原生 KafkaJS 实现](https://kafka.js.org/docs/consuming#manual-committing)一样工作。

```typescript
@@filename()
@EventPattern('user.created')
async handleUserCreated(@Payload() data: IncomingMessage, @Ctx() context: KafkaContext) {
  // 业务逻辑
  
  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  const consumer = context.getConsumer();
  await consumer.commitOffsets([{ topic, partition, offset }])
}
@@switch
@Bind(Payload(), Ctx())
@EventPattern('user.created')
async handleUserCreated(data, context) {
  // 业务逻辑

  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  const consumer = context.getConsumer();
  await consumer.commitOffsets([{ topic, partition, offset }])
}
```

要禁用消息的自动提交，请在 `run` 配置中设置 `autoCommit: false`，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
```