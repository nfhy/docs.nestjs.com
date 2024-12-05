### 拦截器

[常规拦截器](/interceptors)和微服务拦截器之间没有区别。以下示例使用了手动实例化的函数作用域拦截器。就像基于HTTP的应用程序一样，您也可以使用控制器作用域拦截器（即，使用`@UseInterceptors()`装饰器前缀控制器类）。

```typescript
@@filename()
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```