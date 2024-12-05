### 异步提供者

有时，应用程序启动应该延迟，直到一个或多个**异步任务**完成。例如，您可能不想在与数据库建立连接之前就开始接受请求。您可以使用异步提供者来实现这一点。

使用`async/await`与`useFactory`语法可以实现这一点。工厂返回一个`Promise`，工厂函数可以`await`异步任务。Nest会在实例化任何依赖（注入）此类提供者的类之前等待承诺的解决。

```typescript
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}
```

> 信息 **提示** 了解更多自定义提供者语法[点击这里](/fundamentals/custom-providers)。

#### 注入

异步提供者像其他提供者一样通过它们的令牌注入到其他组件中。在上面的例子中，您将使用构造`@Inject('ASYNC_CONNECTION')`。

#### 示例

[TypeORM食谱](/recipes/sql-typeorm)提供了一个更全面的异步提供者示例。