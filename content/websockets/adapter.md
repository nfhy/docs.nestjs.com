### 适配器

WebSockets模块是平台无关的，因此，你可以通过使用`WebSocketAdapter`接口来使用自己的库（甚至是原生实现）。这个接口强制实现以下表中描述的一些方法：

| 方法名称 | 描述 |
| --- | --- |
| `create` | 根据传递的参数创建一个socket实例 |
| `bindClientConnect` | 绑定客户端连接事件 |
| `bindClientDisconnect` | 绑定客户端断开连接事件（可选） |
| `bindMessageHandlers` | 将传入的消息绑定到相应的消息处理器 |
| `close` | 终止一个服务器实例 |

#### 扩展socket.io

[socket.io](https://github.com/socketio/socket.io)包被封装在一个`IoAdapter`类中。如果你想增强适配器的基本功能怎么办？例如，你的技术要求需要在多个负载均衡的web服务实例之间广播事件的能力。为此，你可以扩展`IoAdapter`并覆盖一个方法，该方法的职责是实例化新的socket.io服务器。但首先，让我们安装所需的包。

> 警告：要使用socket.io与多个负载均衡实例，你要么必须在客户端的socket.io配置中设置`transports: ['websocket']`来禁用轮询，要么你必须在负载均衡器中启用基于cookie的路由。仅Redis是不够的。更多信息请[点击这里](https://socket.io/docs/v4/using-multiple-nodes/#enabling-sticky-session)。

```bash
$ npm i --save redis socket.io @socket.io/redis-adapter
```

一旦安装了包，我们可以创建一个`RedisIoAdapter`类。

```typescript
import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  async connectToRedis(): Promise<void> {
    const pubClient = createClient({ url: `redis://localhost:6379` });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

之后，简单地切换到你新创建的Redis适配器。

```typescript
const app = await NestFactory.create(AppModule);
const redisIoAdapter = new RedisIoAdapter(app);
await redisIoAdapter.connectToRedis();

app.useWebSocketAdapter(redisIoAdapter);
```

#### Ws库

另一个可用的适配器是`WsAdapter`，它充当框架和集成快速且经过充分测试的[ws](https://github.com/websockets/ws)库之间的代理。这个适配器与原生浏览器WebSockets完全兼容，并且比socket.io包快得多。不幸的是，它提供的开箱即用功能要少得多。在某些情况下，你可能根本不需要它们。

> 提示：`ws`库不支持命名空间（由`socket.io`推广的通信渠道）。然而，为了在某种程度上模仿这个特性，你可以在不同的路径上挂载多个`ws`服务器（例如：`@WebSocketGateway({ path: '/users' })`）。

为了使用`ws`，我们首先必须安装所需的包：

```bash
$ npm i --save @nestjs/platform-ws
```

一旦安装了包，我们可以切换适配器：

```typescript
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

> 提示：`WsAdapter`是从`@nestjs/platform-ws`导入的。

#### 高级（自定义适配器）

为了演示目的，我们将手动集成[ws](https://github.com/websockets/ws)库。如前所述，这个库的适配器已经创建，并作为`WsAdapter`类从`@nestjs/platform-ws`包中暴露出来。以下是简化实现可能的样子：

```typescript
import * as WebSocket from 'ws';
import { WebSocketAdapter, INestApplicationContext } from '@nestjs/common';
import { MessageMappingProperties } from '@nestjs/websockets';
import { Observable, fromEvent, EMPTY } from 'rxjs';
import { mergeMap, filter } from 'rxjs/operators';

export class WsAdapter implements WebSocketAdapter {
  constructor(private app: INestApplicationContext) {}

  create(port: number, options: any = {}): any {
    return new WebSocket.Server({ port, ...options });
  }

  bindClientConnect(server, callback: Function) {
    server.on('connection', callback);
  }

  bindMessageHandlers(
    client: WebSocket,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ) {
    fromEvent(client, 'message')
      .pipe(
        mergeMap(data => this.bindMessageHandler(data, handlers, process)),
        filter(result => result),
      )
      .subscribe(response => client.send(JSON.stringify(response)));
  }

  bindMessageHandler(
    buffer,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ): Observable<any> {
    const message = JSON.parse(buffer.data);
    const messageHandler = handlers.find(
      handler => handler.message === message.event,
    );
    if (!messageHandler) {
      return EMPTY;
    }
    return process(messageHandler.callback(message.data));
  }

  close(server) {
    server.close();
  }
}
```

> 提示：当你想要利用[ws](https://github.com/websockets/ws)库时，使用内置的`WsAdapter`而不是创建你自己的。

然后，我们可以使用`useWebSocketAdapter()`方法设置自定义适配器：

```typescript
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

#### 示例

使用`WsAdapter`的工作示例可以在[这里](https://github.com/nestjs/nest/tree/master/sample/16-gateways-ws)找到。