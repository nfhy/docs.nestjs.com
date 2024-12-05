### 缓存

缓存是一种伟大而简单的**技术**，它有助于提高应用程序的性能。它充当临时数据存储，提供高性能的数据访问。

#### 安装

首先安装所需的包：

```bash
$ npm install @nestjs/cache-manager cache-manager
```

> 警告 **警告** `cache-manager` 版本 4 使用秒作为 `TTL (Time-To-Live)`。当前版本的 `cache-manager` (v5) 已经切换为使用毫秒。NestJS 不会转换该值，而是直接将您提供的 ttl 传递给库。换句话说：

> 
> - 如果使用 `cache-manager` v4，请以秒为单位提供 ttl
> - 如果使用 `cache-manager` v5，请以毫秒为单位提供 ttl
> - 文档提到的是以秒为单位，因为 NestJS 发布时针对的是 cache-manager 的第 4 版。

#### 内存缓存

Nest 提供了一个统一的 API，用于各种缓存存储提供商。内置的是内存数据存储。然而，您可以轻松切换到更全面的解决方案，如 Redis。

为了启用缓存，导入 `CacheModule` 并调用其 `register()` 方法。

```typescript
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
})
export class AppModule {}
```

#### 与缓存存储交互

要与缓存管理器实例交互，请使用 `CACHE_MANAGER` 令牌将其注入到您的类中，如下所示：

```typescript
constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}
```

> 提示 **提示** `Cache` 类是从 `cache-manager` 导入的，而 `CACHE_MANAGER` 令牌是从 `@nestjs/cache-manager` 包导入的。

`Cache` 实例上的 `get` 方法（来自 `cache-manager` 包）用于从缓存中检索项目。如果缓存中不存在该项目，则返回 `null`。

```typescript
const value = await this.cacheManager.get('key');
```

要向缓存中添加项目，请使用 `set` 方法：

```typescript
await this.cacheManager.set('key', 'value');
```

> 注意 **注意** 内存缓存存储只能存储 [结构化克隆算法](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#javascript_types) 支持的类型。

缓存的默认过期时间是 5 秒。

您可以手动为这个特定键指定一个 TTL（以秒为单位的过期时间），如下所示：

```typescript
await this.cacheManager.set('key', 'value', 1000);
```

要禁用缓存的过期，将 `ttl` 配置属性设置为 `0`：

```typescript
await this.cacheManager.set('key', 'value', 0);
```

要从缓存中删除项目，请使用 `del` 方法：

```typescript
await this.cacheManager.del('key');
```

要清除整个缓存，请使用 `reset` 方法：

```typescript
await this.cacheManager.reset();
```

#### 自动缓存响应

> 警告 **警告** 在 [GraphQL](/graphql/quick-start) 应用程序中，拦截器是为每个字段解析器单独执行的。因此，`CacheModule`（使用拦截器来缓存响应）将无法正常工作。

要启用自动缓存响应，只需在您想要缓存数据的地方绑定 `CacheInterceptor`。

```typescript
@Controller()
@UseInterceptors(CacheInterceptor)
export class AppController {
  @Get()
  findAll(): string[] {
    return [];
  }
}
```

> 警告 **警告** 仅缓存 `GET` 端点。此外，注入原生响应对象（`@Res()`）的 HTTP 服务器路由不能使用缓存拦截器。更多详情请参阅 [响应映射](https://docs.nestjs.com/interceptors#response-mapping)。

为了减少所需的样板代码，您可以将 `CacheInterceptor` 绑定到所有端点全局：

```typescript
import { Module } from '@nestjs/common';
import { CacheModule, CacheInterceptor } from '@nestjs/cache-manager';
import { AppController } from './app.controller';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: CacheInterceptor,
    },
  ],
})
export class AppModule {}
```

#### 自定义缓存

所有缓存数据都有自己的过期时间（[TTL](https://en.wikipedia.org/wiki/Time_to_live)）。要自定义默认值，请将选项对象传递给 `register()` 方法。

```typescript
CacheModule.register({
  ttl: 5, // 秒
  max: 10, // 缓存中的最大项目数
});
```

#### 全局使用模块

当您想要在其他模块中使用 `CacheModule` 时，您需要导入它（像使用任何其他 Nest 模块一样）。或者，通过将选项对象的 `isGlobal` 属性设置为 `true`，声明它为 [全局模块](https://docs.nestjs.com/modules#global-modules)，如下所示。在这种情况下，一旦它在根模块（例如 `AppModule`）中加载，您将不需要在其他模块中导入 `CacheModule`。

```typescript
CacheModule.register({
  isGlobal: true,
});
```

#### 全局缓存覆盖

当启用全局缓存时，缓存条目存储在基于路由路径自动生成的 `CacheKey` 下。您可以在每个控制器方法的基础上覆盖某些缓存设置（`@CacheKey()` 和 `@CacheTTL()`），允许为单个控制器方法定制缓存策略。这在使用 [不同的缓存存储](https://docs.nestjs.com/techniques/caching#different-stores) 时可能最为相关。

您可以在每个控制器的基础上应用 `@CacheTTL()` 装饰器，为整个控制器设置缓存 TTL。在同时定义了控制器级别和方法级别的缓存 TTL 设置的情况下，方法级别上指定的缓存 TTL 设置将优先于在控制器级别上设置的。

```typescript
@Controller()
@CacheTTL(50)
export class AppController {
  @CacheKey('custom_key')
  @CacheTTL(20)
  findAll(): string[] {
    return [];
  }
}
```

> 提示 **提示** `@CacheKey()` 和 `@CacheTTL()` 装饰器是从 `@nestjs/cache-manager` 包导入的。

`@CacheKey()` 装饰器可以与或不与相应的 `@CacheTTL()` 装饰器一起使用，反之亦然。您可以选择仅覆盖 `@CacheKey()` 或仅覆盖 `@CacheTTL()`。未用装饰器覆盖的设置将使用全局注册的默认值（见 [自定义缓存](https://docs.nestjs.com/techniques/caching#customize-caching)）。

#### WebSocket 和微服务

您还可以将 `CacheInterceptor` 应用于 WebSocket 订阅者以及微服务的模式（无论使用何种传输方法）。

```typescript
@@filename()
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
@@switch
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client, data) {
  return [];
}
```

然而，需要额外的 `@CacheKey()` 装饰器来指定用于存储和检索缓存数据的键。另外，请注意，您**不应该缓存一切**。执行一些业务操作的动作，而不仅仅是查询数据的动作，永远不应该被缓存。

此外，您可以使用 `@CacheTTL()` 装饰器指定缓存过期时间（TTL），这将覆盖全局默认的 TTL 值。

```typescript
@@filename()
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
@@switch
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client, data) {
  return [];
}
```

> 提示 **提示** `@CacheTTL()` 装饰器可以与或不与相应的 `@CacheKey()` 装饰器一起使用。

#### 调整跟踪

默认情况下，Nest 使用请求 URL（在 HTTP 应用中）或缓存键（在 WebSocket 和微服务应用中，通过 `@CacheKey()` 装饰器设置）来将缓存记录与您的端点关联起来。然而，有时您可能想要根据不同的因素设置跟踪，例如使用 HTTP 头部（例如 `Authorization` 以正确识别 `profile` 端点）。

为了实现这一点，请创建 `CacheInterceptor` 的子类并覆盖 `trackBy()` 方法。

```typescript
@Injectable()
class HttpCacheInterceptor extends CacheInterceptor {
  trackBy(context: ExecutionContext): string | undefined {
    return 'key';
  }
}
```

#### 不同的存储

`cache-manager` 包提供了多种有用的存储选项，包括 [Redis 存储](https://www.npmjs.com/package/cache-manager-redis-yet)，这是用于将 Redis 与 cache-manager 集成的官方包。您可以在 [这里](https://github.com/jaredwray/cacheable/blob/main/packages/cache-manager/READMEv5.md#store-engines) 找到支持的存储列表。要配置 Redis 存储，请使用 `registerAsync()` 方法进行初始化，如下所示：

```typescript
import { redisStore } from 'cache-manager-redis-yet';
import { Module } from '@nestjs/common';
import { CacheModule, CacheStore } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [
    CacheModule.registerAsync({
      useFactory: async () => {
        const store = await redisStore({
          socket: {
            host: 'localhost',
            port: 6379,
          },
        });

        return {
          store: store as unknown as CacheStore,
          ttl: 3 * 60000, // 3 分钟（毫秒）
        };
      },
    }),
  ],
  controllers: [AppController],
})
export class AppModule {}
```

> 警告 **警告** `cache-manager-redis-yet` 包要求直接指定 `ttl`（生存时间）设置，而不是作为模块选项的一部分。

#### 异步配置

您可能想要异步传递模块选项，而不是在编译时静态传递。在这种情况下，请使用 `registerAsync()` 方法，它提供了几种处理异步配置的方法。

一种方法是使用工厂函数：

```typescript
CacheModule.registerAsync({
  useFactory: () => ({
    ttl: 5,
  }),
});
```

我们的工厂像所有其他异步模块工厂一样（它可以是 `async`，并且能够通过 `inject` 注入依赖项）。

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    ttl: configService.get('CACHE_TTL'),
  }),
  inject: [ConfigService],
});
```

或者，您可以使用 `useClass` 方法：

```typescript
CacheModule.registerAsync({
  useClass: CacheConfigService,
});
```

上述构造将在 `CacheModule` 内实例化 `CacheConfigService` 并使用它来获取选项对象。`CacheConfigService` 必须实现 `CacheOptionsFactory` 接口以提供配置选项：

```typescript
@Injectable()
class CacheConfigService implements CacheOptionsFactory {
  createCacheOptions(): CacheModuleOptions {
    return {
      ttl: 5,
    };
  }
}
```

如果您希望使用从不同模块导入的现有配置提供商，请使用 `useExisting` 语法：

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

这与 `useClass` 相同，有一个关键区别 - `CacheModule` 将查找导入的模块以重用任何已创建的 `ConfigService`，而不是实例化自己的。

> 提示 **提示** `CacheModule#register` 和 `CacheModule#registerAsync` 和 `CacheOptionsFactory` 有一个可选的泛型（类型参数）来缩小特定存储配置选项，使其类型安全。

您还可以将所谓的 `extraProviders` 传递给 `registerAsync()` 方法。这些提供者将与模块提供者合并。

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useClass: ConfigService,
  extraProviders: [MyAdditionalProvider],
});
```

当您想要为工厂函数或类构造函数提供额外的依赖项时，这很有用。

#### 示例

一个工作示例可以在 [这里](https://github.com/nestjs/nest/tree/master/sample/20-cache) 找到。