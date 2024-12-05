### 网关

在本文档中讨论的大多数概念，如依赖注入、装饰器、异常过滤器、管道、守卫和拦截器，同样适用于网关。Nest 抽象了实现细节，使得相同的组件可以在基于 HTTP 的平台、WebSocket 和微服务上运行。本节涵盖了 Nest 中特定于 WebSocket 的方面。

在 Nest 中，网关是一个用 `@WebSocketGateway()` 装饰器注解的类。从技术上讲，网关是平台无关的，这意味着一旦创建了适配器，它们就与任何 WebSockets 库兼容。有两个 WS 平台是开箱即用的：[socket.io](https://github.com/socketio/socket.io) 和 [ws](https://github.com/websockets/ws)。你可以选择最适合你需求的那一个。此外，你可以通过遵循这个[指南](/websockets/adapter)来构建自己的适配器。

<figure><img class="illustrative-image" src="/assets/Gateways_1.png" /></figure>

> **提示**：网关可以被视为[提供者](/providers)；这意味着它们可以通过类构造函数注入依赖项。此外，网关也可以被其他类（提供者和控制器）注入。

#### 安装

要开始构建基于 WebSocket 的应用程序，首先安装所需的包：

```bash
$ npm i --save @nestjs/websockets @nestjs/platform-socket.io
```

#### 概览

通常，每个网关都监听与 **HTTP 服务器** 相同的端口，除非你的应用程序不是 Web 应用程序，或者你手动更改了端口。你可以通过向 `@WebSocketGateway(80)` 装饰器传递一个参数来修改这种默认行为，其中 `80` 是选择的端口号。你还可以使用以下构造来设置网关使用的[命名空间](https://socket.io/docs/v4/namespaces/)：

```typescript
@WebSocketGateway(80, { namespace: 'events' })
```

> **警告**：网关在被引用在现有模块的提供者数组中之前不会被实例化。

你可以将任何支持的[选项](https://socket.io/docs/v4/server-options/)传递给 socket 构造函数，作为 `@WebSocketGateway()` 装饰器的第二个参数，如下所示：

```typescript
@WebSocketGateway(81, { transports: ['websocket'] })
```

网关现在正在监听，但我们还没有订阅任何传入的消息。让我们创建一个处理器，它将订阅 `events` 消息，并用与用户发送的完全相同的数据回应用户。

```typescript
@SubscribeMessage('events')
handleEvent(@MessageBody() data: string): string {
  return data;
}
```

> **提示**：`@SubscribeMessage()` 和 `@MessageBody()` 装饰器是从 `@nestjs/websockets` 包导入的。

一旦创建了网关，我们可以在我们的模块中注册它。

```typescript
import { Module } from '@nestjs/common';
import { EventsGateway } from './events.gateway';

@Module({
  providers: [EventsGateway]
})
export class EventsModule {}
```

你还可以将一个属性键传递给装饰器，以从传入的消息正文中提取它：

```typescript
@SubscribeMessage('events')
handleEvent(@MessageBody('id') id: number): number {
  // id === messageBody.id
  return id;
}
```

如果你更倾向于不使用装饰器，以下代码在功能上是等效的：

```typescript
@SubscribeMessage('events')
handleEvent(client: Socket, data: string): string {
  return data;
}
```

在上面的例子中，`handleEvent()` 函数接受两个参数。第一个是平台特定的[socket 实例](https://socket.io/docs/v4/server-api/#socket)，而第二个是从客户端接收到的数据。这种方法不推荐，因为它需要在每个单元测试中模拟 `socket` 实例。

一旦接收到 `events` 消息，处理器就会发送一个带有与网络发送的相同数据的确认。此外，可以使用库特定的方法发送消息，例如，通过使用 `client.emit()` 方法。为了访问连接的 socket 实例，使用 `@ConnectedSocket()` 装饰器。

```typescript
@SubscribeMessage('events')
handleEvent(
  @MessageBody() data: string,
  @ConnectedSocket() client: Socket,
): string {
  return data;
}
```

> **提示**：`@ConnectedSocket()` 装饰器是从 `@nestjs/websockets` 包导入的。

然而，在这种情况下，你将无法利用拦截器。如果你不想回应用户，你可以简单地跳过 `return` 语句（或者显式返回一个“假”值，例如 `undefined`）。

现在，当客户端如下发送消息时：

```typescript
socket.emit('events', { name: 'Nest' });
```

`handleEvent()` 方法将被执行。为了监听上述处理器内部发出的消息，客户端必须附加相应的确认监听器：

```typescript
socket.emit('events', { name: 'Nest' }, (data) => console.log(data));
```

#### 多个响应

确认只发送一次。此外，它不受原生 WebSockets 实现的支持。为了解决这个限制，你可以返回一个对象，该对象由两个属性组成。`event` 是发出事件的名称，`data` 是必须转发到客户端的数据。

```typescript
@SubscribeMessage('events')
handleEvent(@MessageBody() data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
```

> **提示**：`WsResponse` 接口是从 `@nestjs/websockets` 包导入的。

> **警告**：如果你的 `data` 字段依赖于 `ClassSerializerInterceptor`，你应该返回一个实现 `WsResponse` 的类实例，因为它忽略了普通的 JavaScript 对象响应。

为了监听传入的响应，客户端必须应用另一个事件监听器。

```typescript
socket.on('events', (data) => console.log(data));
```

#### 异步响应

消息处理器能够同步或**异步**地响应。因此，支持 `async` 方法。消息处理器还能够返回一个 `Observable`，在这种情况下，结果值将被发射，直到流完成。

```typescript
@SubscribeMessage('events')
onEvent(@MessageBody() data: unknown): Observable<WsResponse<number>> {
  const event = 'events';
  const response = [1, 2, 3];

  return from(response).pipe(
    map(data => ({ event, data })),
  );
}
```

在上面的例子中，消息处理器将响应 **3次**（数组中的每个项目）。

#### 生命周期钩子

有 3 个有用的生命周期钩子可用。它们都有相应的接口，并在下面的表格中描述：

<table>
  <tr>
    <td>
      <code>OnGatewayInit</code>
    </td>
    <td>
      强制实现 <code>afterInit()</code> 方法。接收库特定的服务器实例作为参数（如果需要，还可以展开其余的）。
    </td>
  </tr>
  <tr>
    <td>
      <code>OnGatewayConnection</code>
    </td>
    <td>
      强制实现 <code>handleConnection()</code> 方法。接收库特定的客户端 socket 实例作为参数。
    </td>
  </tr>
  <tr>
    <td>
      <code>OnGatewayDisconnect</code>
    </td>
    <td>
      强制实现 <code>handleDisconnect()</code> 方法。接收库特定的客户端 socket 实例作为参数。
    </td>
  </tr>
</table>

> **提示**：每个生命周期接口都从 `@nestjs/websockets` 包中暴露出来。

#### 服务器和命名空间

有时，你可能想要直接访问原生的、**平台特定的**服务器实例。这个对象的引用作为参数传递给 `afterInit()` 方法（`OnGatewayInit` 接口）。另一个选项是使用 `@WebSocketServer()` 装饰器。

```typescript
@WebSocketServer()
server: Server;
```

此外，你可以使用 `namespace` 属性检索相应的命名空间，如下所示：

```typescript
@WebSocketServer({ namespace: 'my-namespace' })
namespace: Namespace;
```

> **注意**：`@WebSocketServer()` 装饰器是从 `@nestjs/websockets` 包导入的。

Nest 会在服务器实例准备就绪后自动将服务器实例分配给这个属性。

#### 示例

一个工作示例可在[这里](https://github.com/nestjs/nest/tree/master/sample/02-gateways)找到。