### 全局前缀

要在HTTP应用程序中为**所有路由**设置一个前缀，可以使用`INestApplication`实例的`setGlobalPrefix()`方法。

```typescript
const app = await NestFactory.create(AppModule);
app.setGlobalPrefix('v1');
```

你可以使用以下构造排除全局前缀的路由：

```typescript
app.setGlobalPrefix('v1', {
  exclude: [{ path: 'health', method: RequestMethod.GET }],
});
```

或者，你可以指定一个字符串形式的路由（它将适用于每个请求方法）：

```typescript
app.setGlobalPrefix('v1', { exclude: ['cats'] });
```

> 信息 **提示** `path`属性支持使用[path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters)包的通配符参数。注意：这不接受通配符星号`*`。相反，你必须使用参数（例如，`(.*)`，`:splat*`）。