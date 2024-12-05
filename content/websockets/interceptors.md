### 拦截器

常规拦截器和WebSocket拦截器之间没有区别。以下示例使用了手动实例化的、方法作用域的拦截器。就像基于HTTP的应用程序一样，你也可以使用网关作用域的拦截器（即，在网关类前加上`@UseInterceptors()`装饰器）。

```typescript
@@filename()
@UseInterceptors(new TransformInterceptor())
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UseInterceptors(new TransformInterceptor())
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```