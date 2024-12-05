### 管道

[常规管道](/pipes) 和 WebSocket 管道之间没有根本区别。唯一的区别在于，你应该使用 `WsException` 而不是抛出 `HttpException`。此外，所有的管道只会应用于 `data` 参数（因为验证或转换 `client` 实例是无用的）。

> 信息 **提示** `WsException` 类是从 `@nestjs/websockets` 包中暴露出来的。

#### 绑定管道

以下示例使用了手动实例化的函数作用域管道。就像基于 HTTP 的应用程序一样，你也可以使用网关作用域管道（即，在网关类前加上 `@UsePipes()` 装饰器）。

```typescript
@@filename()
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new WsException(errors) }))
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new WsException(errors) }))
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```