### MQTT

[MQTT](https://mqtt.org/)（消息队列遥测传输）是一个开源的轻量级消息协议，优化了低延迟。该协议提供了一个可扩展且成本效益高的连接设备方式，使用**发布/订阅**模型。基于MQTT的通信系统包括发布服务器、代理服务器和一台或多台客户端。它专为受限设备和低带宽、高延迟或不可靠的网络而设计。

#### 安装

要开始构建基于MQTT的微服务，请先安装所需的包：

```bash
$ npm i --save mqtt
```

#### 概览

要使用MQTT传输器，请将以下选项对象传递给`createMicroservice()`方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
```

> 信息 **提示** `Transport` 枚举是从 `@nestjs/microservices` 包中导入的。

#### 选项

`options` 对象是特定于所选传输器的。`MQTT` 传输器暴露了[这里](https://github.com/mqttjs/MQTT.js/#mqttclientstreambuilder-options)描述的属性。

#### 客户端

像其他微服务传输器一样，您有[多种选项](https://docs.nestjs.com/microservices/basics#client)来创建MQTT `ClientProxy` 实例。

使用`ClientsModule`创建客户端实例的一种方法是导入它，并使用`register()`方法传递一个选项对象，该对象具有与`createMicroservice()`方法中上面显示的相同属性，以及一个`name`属性，用作注入令牌。更多关于`ClientsModule`的信息[在这里](https://docs.nestjs.com/microservices/basics#client)。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.MQTT,
        options: {
          url: 'mqtt://localhost:1883',
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他创建客户端的选项（无论是`ClientProxyFactory`还是`@Client()`）。您可以[在这里](https://docs.nestjs.com/microservices/basics#client)阅读更多信息。

#### 上下文

在更复杂的情况下，您可能想要访问更多关于传入请求的信息。当使用MQTT传输器时，您可以访问`MqttContext`对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

> 信息 **提示** `@Payload()`, `@Ctx()` 和 `MqttContext` 是从 `@nestjs/microservices` 包中导入的。

要访问原始的mqtt [包](https://github.com/mqttjs/mqtt-packet)，请使用`MqttContext`对象的`getPacket()`方法，如下所示：

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(context.getPacket());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getPacket());
}
```

#### 通配符

订阅可能是明确的主题，或者可能包含通配符。有两种通配符可用，`+`和`#`。`+`是单级通配符，而`#`是多级通配符，涵盖许多主题级别。

```typescript
@@filename()
@MessagePattern('sensors/+/temperature/+')
getTemperature(@Ctx() context: MqttContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Ctx())
@MessagePattern('sensors/+/temperature/+')
getTemperature(context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

#### 服务质量 (QoS)

任何使用`@MessagePattern`或`@EventPattern`装饰器创建的订阅都将使用QoS 0进行订阅。如果需要更高的QoS，可以如下在全球范围内使用`subscribeOptions`块建立连接时设置：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
    subscribeOptions: {
      qos: 2
    },
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
    subscribeOptions: {
      qos: 2
    },
  },
});
```

如果需要特定于主题的QoS，请考虑创建[自定义传输器](https://docs.nestjs.com/microservices/custom-transport)。

#### 记录构建器

要配置消息选项（调整QoS级别，设置Retain或DUP标志，或向有效载荷添加其他属性），您可以使用`MqttRecordBuilder`类。例如，要将`QoS`设置为`2`，请使用`setQoS`方法，如下所示：

```typescript
const userProperties = { 'x-version': '1.0.0' };
const record = new MqttRecordBuilder(':cat:')
  .setProperties({ userProperties })
  .setQoS(1)
  .build();
client.send('replace-emoji', record).subscribe(...);
```

> 信息 **提示** `MqttRecordBuilder` 类是从 `@nestjs/microservices` 包中导出的。

您也可以通过访问`MqttContext`在服务器端读取这些选项。

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: MqttContext): string {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

在某些情况下，您可能想要为多个请求配置用户属性，您可以将这些选项传递给`ClientProxyFactory`。

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.MQTT,
          options: {
            url: 'mqtt://localhost:1833',
            userProperties: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```