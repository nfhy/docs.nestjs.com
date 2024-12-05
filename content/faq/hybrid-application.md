混合应用程序
混合应用程序是能够监听来自两个或更多不同源的请求的应用程序。它可以将HTTP服务器与微服务监听器结合，甚至只是多个不同的微服务监听器。默认的 `createMicroservice` 方法不允许多个服务器，因此在这种情况下，每个微服务都必须手动创建和启动。为了实现这一点，可以通过 `connectMicroservice()` 方法将 `INestApplication` 实例与 `INestMicroservice` 实例连接起来。

```typescript
const app = await NestFactory.create(AppModule);
const microservice = app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.TCP,
});

await app.startAllMicroservices();
await app.listen(3001);
```

> 信息提示 **提示** `app.listen(port)` 方法在指定的地址上启动HTTP服务器。如果您的应用程序不处理HTTP请求，则应该使用 `app.init()` 方法代替。

要连接多个微服务实例，为每个微服务发出 `connectMicroservice()` 调用：

```typescript
const app = await NestFactory.create(AppModule);
// 微服务 #1
const microserviceTcp = app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.TCP,
  options: {
    port: 3001,
  },
});
// 微服务 #2
const microserviceRedis = app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});

await app.startAllMicroservices();
await app.listen(3001);
```

要在具有多个微服务的混合应用程序中，仅将 `@MessagePattern()` 绑定到一种传输策略（例如，MQTT），我们可以传递第二个参数类型为 `Transport`，这是一个枚举，定义了所有内置的传输策略。

```typescript
@@filename()
@MessagePattern('time.us.*', Transport.NATS)
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@MessagePattern({ cmd: 'time.us' }, Transport.TCP)
getTCPDate(@Payload() data: number[]) {
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*', Transport.NATS)
getDate(data, context) {
  console.log(`Subject: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@Bind(Payload(), Ctx())
@MessagePattern({ cmd: 'time.us' }, Transport.TCP)
getTCPDate(data, context) {
  return new Date().toLocaleTimeString(...);
}
```

> 信息提示 `@Payload()`, `@Ctx()`, `Transport` 和 `NatsContext` 是从 `@nestjs/microservices` 导入的。

#### 共享配置
默认情况下，混合应用程序不会继承为主（基于HTTP）应用程序配置的全局管道、拦截器、守卫和过滤器。
要从主应用程序继承这些配置属性，在 `connectMicroservice()` 调用的第二个参数（一个可选的选项对象）中设置 `inheritAppConfig` 属性，如下所示：

```typescript
const microservice = app.connectMicroservice<MicroserviceOptions>(
  {
    transport: Transport.TCP,
  },
  {
    inheritAppConfig: true,
  },
);
```