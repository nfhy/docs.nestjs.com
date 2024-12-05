### HTTP 适配器

有时，您可能希望在 Nest 应用程序上下文内或外部访问底层 HTTP 服务器。

每个原生（平台特定）HTTP 服务器/库（例如，Express 和 Fastify）实例都被封装在一个**适配器**中。适配器被注册为一个全局可用的提供者，可以从应用程序上下文中检索，也可以注入到其他提供者中。

#### 应用上下文外的策略

要从应用程序上下文外部获取对 `HttpAdapter` 的引用，请调用 `getHttpAdapter()` 方法。

```typescript
const app = await NestFactory.create(AppModule);
const httpAdapter = app.getHttpAdapter();
```

#### 作为可注入的

要从应用程序上下文中获取对 `HttpAdapterHost` 的引用，使用与其他现有提供者相同的技术（例如，使用构造函数注入）进行注入。

```typescript
export class CatsService {
  constructor(private adapterHost: HttpAdapterHost) {}
}
```

```typescript
@Dependencies(HttpAdapterHost)
export class CatsService {
  constructor(adapterHost) {
    this.adapterHost = adapterHost;
  }
}
```

> 信息 **提示** `HttpAdapterHost` 是从 `@nestjs/core` 包导入的。

`HttpAdapterHost` **不是** 真正的 `HttpAdapter`。要获取实际的 `HttpAdapter` 实例，只需访问 `httpAdapter` 属性。

```typescript
const adapterHost = app.get(HttpAdapterHost);
const httpAdapter = adapterHost.httpAdapter;
```

`httpAdapter` 是底层框架使用的 HTTP 适配器的实际实例。它是 `ExpressAdapter` 或 `FastifyAdapter` 的一个实例（这两个类都扩展了 `AbstractHttpAdapter`）。

适配器对象公开了几个有用的方法来与 HTTP 服务器交互。然而，如果您想直接访问库实例（例如，Express 实例），请调用 `getInstance()` 方法。

```typescript
const instance = httpAdapter.getInstance();
```

#### 监听事件

要执行服务器开始监听传入请求时的操作，可以订阅 `listen$` 流，如下所示：

```typescript
this.httpAdapterHost.listen$.subscribe(() => 
  console.log('HTTP server is listening'),
);
```

此外，`HttpAdapterHost` 提供了一个 `listening` 布尔属性，指示服务器当前是否处于活动状态并正在监听：

```typescript
if (this.httpAdapterHost.listening) {
  console.log('HTTP server is listening');
}
```