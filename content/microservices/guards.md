### 守卫

微服务守卫和[常规HTTP应用程序守卫](/guards)之间没有根本区别。
唯一的区别在于，你应该使用`RpcException`而不是抛出`HttpException`。

> 信息 **提示** `RpcException`类是从`@nestjs/microservices`包中导出的。

#### 绑定守卫

以下示例使用了一个方法作用域的守卫。就像基于HTTP的应用程序一样，你也可以使用控制器作用域的守卫（即，用`@UseGuards()`装饰器前缀控制器类）。

```typescript
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
```

```typescript
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```