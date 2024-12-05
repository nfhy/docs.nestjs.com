### 管道

在[常规管道](/pipes)和微服务管道之间没有根本的区别。唯一的区别在于，你应该使用`RpcException`而不是抛出`HttpException`。

> 信息 **提示** `RpcException`类是从`@nestjs/microservices`包中暴露出来的。

#### 绑定管道

以下示例使用手动实例化的函数作用域管道。就像基于HTTP的应用程序一样，你也可以使用控制器作用域管道（即，用`@UsePipes()`装饰器前缀控制器类）。

```typescript
@@filename()
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new RpcException(errors) }))
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new RpcException(errors) }))
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```