### 守卫

WebSocket 守卫与[常规 HTTP 应用守卫](/guards)之间没有根本区别。唯一的区别在于，你应该使用 `WsException` 而不是抛出 `HttpException`。

> 信息 **提示** `WsException` 类是从 `@nestjs/websockets` 包中导出的。

#### 绑定守卫

以下示例使用了方法作用域的守卫。就像基于 HTTP 的应用程序一样，你也可以使用网关作用域的守卫（即，用 `@UseGuards()` 装饰器前缀网关类）。

```typescript
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
```

```typescript
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```