### 异常过滤器

HTTP [异常过滤器](/exception-filters)层与相应的WebSockets层之间的唯一区别在于，你应该使用`WsException`而不是抛出`HttpException`。

```typescript
throw new WsException('无效的凭证。');
```

> 信息 **提示** `WsException`类是从`@nestjs/websockets`包导入的。

以上示例中，Nest将处理抛出的异常，并发出以下结构的`exception`消息：

```typescript
{
  status: 'error',
  message: '无效的凭证。'
}
```

#### 过滤器

WebSockets异常过滤器的行为与HTTP异常过滤器相同。以下示例使用手动实例化的方法作用域过滤器。就像基于HTTP的应用程序一样，你也可以使用网关作用域过滤器（即，用`@UseFilters()`装饰器前缀网关类）。

```typescript
@UseFilters(new WsExceptionFilter())
@SubscribeMessage('events')
onEvent(client, data: any): WsResponse<any> {
  const event = 'events';
  return { event, data };
}
```

#### 继承

通常情况下，你会创建完全定制化的异常过滤器，以满足你的应用程序需求。然而，可能存在某些情况，当你想要简单地扩展**核心异常过滤器**，并根据某些因素覆盖行为。

为了将异常处理委托给基过滤器，你需要扩展`BaseWsExceptionFilter`并调用继承的`catch()`方法。

```typescript
@Catch()
export class AllExceptionsFilter extends BaseWsExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

以上实现只是一个展示方法的示例。你扩展的异常过滤器的实现将包括你定制的**业务逻辑**（例如，处理各种条件）。