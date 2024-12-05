### 服务器发送事件

服务器发送事件（Server-Sent Events, SSE）是一种服务器推送技术，允许客户端通过HTTP连接自动接收来自服务器的更新。每个通知作为一块文本发送，并以一对换行符终止（了解更多[点击这里](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)）。

#### 使用方法

要在控制器类中注册的路由上启用服务器发送事件，使用`@Sse()`装饰器标注方法处理器。

```typescript
@Sse('sse')
sse(): Observable<MessageEvent> {
  return interval(1000).pipe(map((_) => ({ data: { hello: 'world' } })));
}
```

> 信息提示 **提示** `@Sse()`装饰器和`MessageEvent`接口是从`@nestjs/common`导入的，而`Observable`、`interval`和`map`是从`rxjs`包导入的。

> 警告 **警告** 服务器发送事件路由必须返回一个`Observable`流。

在上面的例子中，我们定义了一个名为`sse`的路由，允许我们传播实时更新。这些事件可以使用[EventSource API](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)来监听。

`sse`方法返回一个`Observable`，它发出多个`MessageEvent`（在这个例子中，它每秒发出一个新的`MessageEvent`）。`MessageEvent`对象应该遵循以下接口以符合规范：

```typescript
export interface MessageEvent {
  data: string | object;
  id?: string;
  type?: string;
  retry?: number;
}
```

有了这个设置，我们现在可以在客户端应用程序中创建一个`EventSource`类的实例，将`/sse`路由（与我们上面传入`@Sse()`装饰器的端点匹配）作为构造函数参数传递。

`EventSource`实例打开了一个到HTTP服务器的持久连接，该服务器以`text/event-stream`格式发送事件。连接保持开启状态，直到通过调用`EventSource.close()`关闭。

一旦连接打开，来自服务器的传入消息以事件的形式传递到你的代码中。如果传入消息中有事件字段，触发的事件与事件字段值相同。如果没有事件字段存在，则触发一个通用的`message`事件（[来源](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)）。

```javascript
const eventSource = new EventSource('/sse');
eventSource.onmessage = ({ data }) => {
  console.log('New message', JSON.parse(data));
};
```

#### 示例

一个工作示例可在[这里](https://github.com/nestjs/nest/tree/master/sample/28-sse)找到。