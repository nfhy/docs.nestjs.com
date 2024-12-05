### 自定义传输器

Nest 提供了多种**传输器**，同时提供了一个 API，允许开发者构建新的自定义传输策略。传输器使您能够通过网络使用可插拔的通信层和非常简单的应用级消息协议连接组件（阅读完整[文章](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3)）。

> 信息 **提示** 构建一个使用 Nest 的微服务并不意味着您必须使用 `@nestjs/microservices` 包。例如，如果您想要与外部服务通信（假设是使用不同语言编写的其他微服务），您可能不需要 `@nestjs/microservice` 库提供的所有特性。

实际上，如果您不需要装饰器（`@EventPattern` 或 `@MessagePattern`），这些装饰器允许您声明性地定义订阅者，运行一个[独立应用程序](/application-context)并手动维护连接/订阅到频道应该对大多数用例来说已经足够，并且会为您提供更多的灵活性。

使用自定义传输器，您可以集成任何消息系统/协议（包括 Google Cloud Pub/Sub、Amazon Kinesis 等）或扩展现有的传输器，添加额外的特性（例如，为 MQTT 添加[QoS](https://github.com/mqttjs/MQTT.js/blob/master/README.md#qos)）。

> 信息 **提示** 为了更好地理解 Nest 微服务的工作原理以及如何扩展现有传输器的能力，我们推荐阅读 [NestJS Microservices in Action](https://dev.to/johnbiundo/series/4724) 和 [Advanced NestJS Microservices](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l) 文章系列。

#### 创建策略

首先，我们定义一个代表我们自定义传输器的类。

```typescript
import { CustomTransportStrategy, Server } from '@nestjs/microservices';

class GoogleCloudPubSubServer
  extends Server
  implements CustomTransportStrategy {
  /**
   * 这个方法在您运行 "app.listen()" 时被触发。
   */
  listen(callback: () => void) {
    callback();
  }

  /**
   * 这个方法在应用程序关闭时被触发。
   */
  close() {}
}
```

> 警告 **警告** 请注意，我们不会在本章节中实现一个全功能的 Google Cloud Pub/Sub 服务器，因为这将需要深入到传输器特定的技术细节。

在上面的例子中，我们声明了 `GoogleCloudPubSubServer` 类，并提供了由 `CustomTransportStrategy` 接口强制的 `listen()` 和 `close()` 方法。此外，我们的类扩展了从 `@nestjs/microservices` 包导入的 `Server` 类，该类提供了一些有用的方法，例如，Nest 运行时用于注册消息处理程序的方法。或者，如果您想要扩展现有传输策略的能力，您可以扩展相应的服务器类，例如 `ServerRedis`。
按照惯例，我们在类名后添加了 `\"Server\"` 后缀，因为它将负责订阅消息/事件（并在需要时响应它们）。

有了这个，我们现在可以使用我们的自定义策略而不是使用内置的传输器，如下所示：

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    strategy: new GoogleCloudPubSubServer(),
  },
);
```

基本上，我们不是传递带有 `transport` 和 `options` 属性的正常传输器选项对象，而是传递一个属性，`strategy`，其值是我们自定义传输器类的实例。

回到我们的 `GoogleCloudPubSubServer` 类，在一个真实的应用程序中，我们将在 `listen()` 方法中建立与我们的消息代理/外部服务的连接并注册订阅者/监听特定频道（然后在 `close()` 清理方法中移除订阅并关闭连接），但由于这需要很好地了解 Nest 微服务如何相互通信，我们推荐阅读这个[文章系列](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l)。
在本章中，我们将专注于 `Server` 类提供的功能以及如何利用它们构建自定义策略。

例如，假设在我们的应用程序中的某个地方定义了以下消息处理程序：

```typescript
@MessagePattern('echo')
echo(@Payload() data: object) {
  return data;
}
```

这个消息处理程序将由 Nest 运行时自动注册。有了 `Server` 类，您可以看到注册了哪些消息模式，并且可以访问并执行实际分配给它们的方法。
为了测试这一点，让我们在 `listen()` 方法中添加一个简单的 `console.log`，在调用 `callback` 函数之前：

```typescript
listen(callback: () => void) {
  console.log(this.messageHandlers);
  callback();
}
```

在您的应用程序重新启动后，您将在终端看到以下日志：

```typescript
Map { 'echo' => [AsyncFunction] { isEventHandler: false } }
```

> 信息 **提示** 如果我们使用了 `@EventPattern` 装饰器，您将看到相同的输出，但是 `isEventHandler` 属性设置为 `true`。

如您所见，`messageHandlers` 属性是一个 `Map` 集合，包含所有消息（和事件）处理程序，其中模式被用作键。
现在，您可以使用键（例如 `\"echo\"`）来接收消息处理程序的引用：

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  console.log(await echoHandler('Hello world!'));
  callback();
}
```

一旦我们执行 `echoHandler` 并传递一个任意字符串作为参数（这里的 `\"Hello world!\"`），我们应该在控制台看到它：

```json
Hello world!
```

这意味着我们的方法处理程序被正确执行了。

当使用带有 [Interceptors](/interceptors) 的 `CustomTransportStrategy` 时，处理程序被包装在 RxJS 流中。这意味着您需要订阅它们以执行流的底层逻辑（例如，在拦截器执行后继续进入控制器逻辑）。

下面是一个例子：

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  const streamOrResult = await echoHandler('Hello World');
  if (isObservable(streamOrResult)) {
    streamOrResult.subscribe();
  }
  callback();
}
```

#### 客户端代理

正如我们在第一部分提到的，您不一定需要使用 `@nestjs/microservices` 包来创建微服务，但如果您决定这样做并且需要集成自定义策略，您将需要提供一个“客户端”类。

> 信息 **提示** 再次强调，实现一个与所有 `@nestjs/microservices` 特性（例如，流控制）兼容的全功能客户端类需要很好地了解框架使用的通信技术。要了解更多信息，请查看这篇文章[文章](https://dev.to/nestjs/part-4-basic-client-component-16f9)。

要与外部服务通信/发送和发布消息（或事件），您可以使用特定库的 SDK 包，或者实现一个扩展 `ClientProxy` 的自定义客户端类，如下所示：

```typescript
import { ClientProxy, ReadPacket, WritePacket } from '@nestjs/microservices';

class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {}
  async close() {}
  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {}
  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {}
}
```

> 警告 **警告** 请注意，我们不会在本章节中实现一个全功能的 Google Cloud Pub/Sub 客户端，因为这将需要深入到传输器特定的技术细节。

如您所见，`ClientProxy` 类要求我们提供几个方法，用于建立和关闭连接以及发布消息（`publish`）和事件（`dispatchEvent`）。
注意，如果您不需要支持请求-响应通信风格，您可以让 `publish()` 方法保持为空。同样，如果您不需要支持基于事件的通信，跳过 `dispatchEvent()` 方法。

为了观察这些方法何时被执行，让我们添加多个 `console.log` 调用，如下所示：

```typescript
class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {
    console.log('connect');
  }

  async close() {
    console.log('close');
  }

  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {
    return console.log('event to dispatch: ', packet);
  }

  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {
    console.log('message:', packet);

    // 在一个真实的应用程序中，\"callback\" 函数应该用从响应器返回的负载执行。
    // 这里，我们简单地模拟（5 秒延迟）响应已经通过，通过传递与我们最初传递的相同的 \"data\"。
    setTimeout(() => callback({ response: packet.data }), 5000);

    return () => console.log('teardown');
  }
}
```

有了这个，让我们创建一个 `GoogleCloudPubSubClient` 类的实例并运行 `send()` 方法（您可能在前面的章节中已经看到过），订阅返回的可观察流。

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .subscribe((response) => console.log(response));
```

现在，您应该在终端看到以下输出：

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
Hello world! // <-- 5 秒后
```

为了测试我们的“teardown”方法（我们的 `publish()` 方法返回的）是否被正确执行，让我们对我们的流应用一个超时操作符，将其设置为 2 秒，以确保它比我们的 `setTimeout` 调用 `callback` 函数更早地抛出错误。

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .pipe(timeout(2000))
  .subscribe(
    (response) => console.log(response),
    (error) => console.error(error.message),
  );
```

> 信息 **提示** `timeout` 操作符是从 `rxjs/operators` 包导入的。

应用了 `timeout` 操作符后，您的终端输出应该如下所示：

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
teardown // <-- 清理
Timeout has occurred
```

要派发一个事件（而不是发送消息），使用 `emit()` 方法：

```typescript
googlePubSubClient.emit('event', 'Hello world!');
```

您应该在控制台看到以下内容：

```typescript
connect
event to dispatch:  { pattern: 'event', data: 'Hello world!' }
```

#### 消息序列化

如果您需要在客户端对响应的序列化添加一些自定义逻辑，您可以使用一个扩展 `ClientProxy` 类或其子类的自定义类。对于修改成功的请求，您可以重写 `serializeResponse` 方法，对于修改通过此客户端传递的任何错误，您可以重写 `serializeError` 方法。要使用这个自定义类，您可以将类本身传递给 `ClientsModule.register()` 方法，使用 `customClass` 属性。下面是一个自定义 `ClientProxy` 的例子，它将每个错误序列化为 `RpcException`。

```typescript
@@filename(error-handling.proxy)
import { ClientTcp, RpcException } from '@nestjs/microservices';

class ErrorHandlingProxy extends ClientTCP {
  serializeError(err: Error) {
    return new RpcException(err);
  }
}
```

然后在 `ClientsModule` 中这样使用它：

```typescript
@@filename(app.module)
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'CustomProxy',
        customClass: ErrorHandlingProxy,
      },
    ]),
  ]
})
export class AppModule {}
```

> 信息 **提示** 这是类本身被传递给 `customClass`，而不是类的实例。Nest 将在内部为您创建实例，并将任何给定的选项传递给新的 `ClientProxy`。