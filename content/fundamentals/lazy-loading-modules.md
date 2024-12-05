### 延迟加载模块

默认情况下，模块是急切加载的，这意味着一旦应用程序加载，所有模块也会随之加载，无论它们是否立即需要。虽然这对大多数应用程序来说都很好，但对于在**无服务器环境**中运行的应用程序/工作器来说，可能会成为瓶颈，因为启动延迟（“冷启动”）至关重要。

延迟加载可以帮助减少启动时间，只加载特定无服务器函数调用所需的模块。此外，您还可以在无服务器函数“热”后异步加载其他模块，以进一步加快后续调用的启动时间（延迟模块注册）。

> **提示**：如果您熟悉**[Angular](https://angular.dev/)**框架，您可能之前已经见过“[延迟加载模块](https://angular.dev/guide/ngmodules/lazy-loading#lazy-loading-basics)”这个术语。请注意，这个技术在Nest中是**功能不同的**，所以将其视为一个完全不同的功能，只是共享了类似的命名约定。

> **警告**：请注意，[生命周期钩子方法](https://docs.nestjs.com/fundamentals/lifecycle-events)在延迟加载的模块和服务中不会被调用。

#### 开始使用

要按需加载模块，Nest提供了`LazyModuleLoader`类，可以像正常一样注入到类中：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}
}
@@switch
@Injectable()
@Dependencies(LazyModuleLoader)
export class CatsService {
  constructor(lazyModuleLoader) {
    this.lazyModuleLoader = lazyModuleLoader;
  }
}
```

> **提示**：`LazyModuleLoader`类是从`@nestjs/core`包导入的。

或者，您可以在应用程序启动文件（`main.ts`）中获得`LazyModuleLoader`提供者的引用，如下所示：

```typescript
// "app"代表Nest应用程序实例
const lazyModuleLoader = app.get(LazyModuleLoader);
```

有了这个，您现在可以使用以下构造来加载任何模块：

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);
```

> **提示**：“延迟加载”的模块在第一次`LazyModuleLoader#load`方法调用后会被**缓存**。这意味着，每次连续尝试加载`LazyModule`都会**非常快**，并且会返回一个缓存的实例，而不是重新加载模块。
>
> ```bash
> > 加载“LazyModule”尝试：1
> > 时间：2.379ms
> > 加载“LazyModule”尝试：2
> > 时间：0.294ms
> > 加载“LazyModule”尝试：3
> > 时间：0.303ms
> ```
>
> 此外，“延迟加载”的模块与在应用程序启动时急切加载的模块以及后来在您的应用程序中注册的任何其他延迟模块共享相同的模块图。

其中`lazy.module.ts`是一个TypeScript文件，它导出了一个**常规的Nest模块**（不需要额外的更改）。

`LazyModuleLoader#load`方法返回[模块引用](/fundamentals/module-ref)（`LazyModule`的），让您可以浏览内部提供者列表，并使用其注入标记作为查找键来获取任何提供者的引用。

例如，假设我们有一个如下定义的`LazyModule`：

```typescript
@Module({
  providers: [LazyService],
  exports: [LazyService],
})
export class LazyModule {}
```

> **提示**：延迟加载模块不能注册为**全局模块**，因为这根本没有意义（因为它们是按需注册的，当所有静态注册的模块已经被实例化后）。同样，注册的**全局增强器**（守卫/拦截器等）**也不会正常工作**。

有了这个，我们可以如下获得`LazyService`提供者的引用：

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);

const { LazyService } = await import('./lazy.service');
const lazyService = moduleRef.get(LazyService);
```

> **警告**：如果您使用**Webpack**，请确保更新您的`tsconfig.json`文件 - 将`compilerOptions.module`设置为`"esnext"`，并添加`compilerOptions.moduleResolution`属性，值为`"node"`：
>
> ```json
> {
>   "compilerOptions": {
>     "module": "esnext",
>     "moduleResolution": "node",
>     ...
>   }
> }
> ```
>
> 有了这些选项设置，您将能够利用[代码分割](https://webpack.js.org/guides/code-splitting/)功能。

#### 延迟加载控制器、网关和解析器

由于控制器（或GraphQL应用程序中的解析器）在Nest中代表一组路由/路径/主题（或查询/突变），您**不能**使用`LazyModuleLoader`类来延迟加载它们。

> **警告**：在延迟加载模块中注册的控制器、[解析器](/graphql/resolvers)和[网关](/websockets/gateways)将不会按预期工作。同样，您也不能按需注册中间件函数（通过实现`MiddlewareConsumer`接口）。

例如，假设您正在使用Fastify驱动程序构建REST API（HTTP应用程序）（使用`@nestjs/platform-fastify`包）。Fastify不允许在应用程序准备就绪/成功监听消息后注册路由。这意味着即使我们分析了模块控制器中注册的路由映射，所有延迟加载的路由都将无法访问，因为没有办法在运行时注册它们。

同样，我们作为`@nestjs/microservices`包一部分提供的某些传输策略（包括Kafka、gRPC或RabbitMQ）需要在建立连接之前订阅/监听特定主题/频道。一旦您的应用程序开始监听消息，框架将无法订阅/监听新主题。

最后，`@nestjs/graphql`包与代码优先方法启用后，会根据元数据自动生成GraphQL模式。这意味着，它需要所有类事先加载。否则，就无法创建适当的、有效的模式。

#### 常见用例

最常见的情况下，您会看到延迟加载模块在您的工作器/定时任务/Lambda & 无服务器函数/网络钩子必须根据输入参数（路由路径/日期/查询参数等）触发不同服务（不同逻辑）时。另一方面，对于启动时间不太重要的单体应用程序来说，延迟加载模块可能没有太多意义。