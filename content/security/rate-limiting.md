### 限流

保护应用程序免受暴力攻击的常用技术是**限流**。要开始使用，你需要安装`@nestjs/throttler`包。

```bash
$ npm i --save @nestjs/throttler
```

安装完成后，`ThrottlerModule`可以像其他Nest包一样使用`forRoot`或`forRootAsync`方法进行配置。

```typescript
@@filename(app.module)
@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        ttl: 60000,
        limit: 10,
      },
    ]),
  ],
})
export class AppModule {}
```

上述配置将为你的应用程序中受保护的路由设置全局选项，`ttl`是存活时间（以毫秒为单位），`limit`是在ttl内的最大请求次数。

一旦导入了模块，你可以选择如何绑定`ThrottlerGuard`。任何在[guards](https://docs.nestjs.com/guards)部分提到的绑定方式都可以。如果你想全局绑定守卫，例如，你可以通过向任何模块添加此提供者来实现：

```typescript
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard
}
```

#### 多限流定义

有时你可能想要设置多个限流定义，比如每秒不超过3个调用，10秒内不超过20个调用，1分钟内不超过100个调用。为此，你可以在数组中设置命名选项，稍后可以在`@SkipThrottle()`和`@Throttle()`装饰器中引用这些选项以更改设置。

```typescript
@@filename(app.module)
@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,
        limit: 3,
      },
      {
        name: 'medium',
        ttl: 10000,
        limit: 20
      },
      {
        name: 'long',
        ttl: 60000,
        limit: 100
      }
    ]),
  ],
})
export class AppModule {}
```

#### 自定义

有时你可能想要将守卫绑定到控制器或全局，但又想禁用一个或多个端点的速率限制。为此，你可以使用`@SkipThrottle()`装饰器，以否定整个类或单个路由的节流器。`@SkipThrottle()`装饰器还可以接受一个对象，其字符串键和布尔值用于如果你想要排除大多数控制器的情况，但又不想排除每个路由，并根据你拥有的多个节流器设置进行配置。如果没有传递对象，默认使用`{ default: true }`。

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {}
```

这个`@SkipThrottle()`装饰器可以用来跳过路由或类，或者在跳过的类中否定路由的跳过。

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {
  // 此路由将应用速率限制。
  @SkipThrottle({ default: false })
  dontSkip() {
    return 'List users work with Rate limiting.';
  }
  // 此路由将跳过速率限制。
  doSkip() {
    return 'List users work without Rate limiting.';
  }
}
```

还有`@Throttle()`装饰器，可以用来覆盖全局模块中设置的`limit`和`ttl`，以提供更严格或更宽松的安全选项。这个装饰器也可以用于类或函数。从版本5开始，装饰器接受一个对象，其中包含与节流器设置名称相关的字符串，以及具有限制和ttl键和整数值的对象，类似于传递给根模块的选项。如果你在原始选项中没有设置名称，请使用字符串`default`。你需要这样配置：

```typescript
// 覆盖默认配置以进行速率限制和持续时间。
@Throttle({ default: { limit: 3, ttl: 60000 } })
@Get()
findAll() {
  return "List users works with custom rate limiting.";
}
```

#### 代理

如果你的应用程序在代理服务器后面运行，配置HTTP适配器以信任代理至关重要。你可以参阅[Express](http://expressjs.com/en/guide/behind-proxies.html)和[Fastify](https://www.fastify.io/docs/latest/Reference/Server/#trustproxy)的特定HTTP适配器选项来启用`trust proxy`设置。

以下示例展示了如何为Express适配器启用`trust proxy`：

```typescript
@@filename(main.ts)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { NestExpressApplication } from '@nestjs/platform-express';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  app.set('trust proxy', 'loopback'); // 信任来自环回地址的请求
  await app.listen(3000);
}

bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { NestExpressApplication } from '@nestjs/platform-express';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.set('trust proxy', 'loopback'); // 信任来自环回地址的请求
  await app.listen(3000);
}

bootstrap();
```

启用`trust proxy`允许你从`X-Forwarded-For`头中检索原始IP地址。你还可以覆盖`getTracker()`方法的行为，从这个头中提取IP地址，而不是依赖于`req.ip`。以下示例展示了如何为Express和Fastify实现这一点：

```typescript
@@filename(throttler-behind-proxy.guard)
import { ThrottlerGuard } from '@nestjs/throttler';
import { Injectable } from '@nestjs/common';

@Injectable()
export class ThrottlerBehindProxyGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    return req.ips.length ? req.ips[0] : req.ip; // 个性化IP提取以满足你自己的需求
  }
}
```

> 信息提示：你可以在[这里](https://expressjs.com/en/api.html#req.ips)找到express的`req`请求对象的API，以及在[这里](https://www.fastify.io/docs/latest/Reference/Request/)找到fastify的。

#### WebSockets

这个模块可以与WebSockets一起工作，但这需要一些类扩展。你可以扩展`ThrottlerGuard`并覆盖`handleRequest`方法，如下所示：

```typescript
@Injectable()
export class WsThrottlerGuard extends ThrottlerGuard {
  async handleRequest(requestProps: ThrottlerRequest): Promise<boolean> {
    const {
      context,
      limit,
      ttl,
      throttler,
      blockDuration,
      getTracker,
      generateKey,
    } = requestProps;

    const client = context.switchToWs().getClient();
    const tracker = client._socket.remoteAddress;
    const key = generateKey(context, tracker, throttler.name);
    const { totalHits, timeToExpire, isBlocked, timeToBlockExpire } =
      await this.storageService.increment(
        key,
        ttl,
        limit,
        blockDuration,
        throttler.name,
      );

    const getThrottlerSuffix = (name: string) =>
      name === 'default' ? '' : `-${ name }`;

    // 当用户达到限制时抛出错误。
    if (isBlocked) {
      await this.throwThrottlingException(context, {
        limit,
        ttl,
        key,
        tracker,
        totalHits,
        timeToExpire,
        isBlocked,
        timeToBlockExpire,
      });
    }

    return true;
  }
}
```

> 信息提示：如果你使用的是ws，需要将`_socket`替换为`conn`。

使用WebSockets时需要注意以下几点：

- 守卫不能使用`APP_GUARD`或`app.useGlobalGuards()`注册
- 当达到限制时，Nest将发出一个`exception`事件，因此请确保有一个监听器准备好处理这个事件

> 信息提示：如果你使用的是`@nestjs/platform-ws`包，你可以使用`client._socket.remoteAddress`代替。

#### GraphQL

`ThrottlerGuard`也可以用于GraphQL请求。再次，守卫需要扩展，但这次是覆盖`getRequestResponse`方法。

```typescript
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gqlCtx.getContext();
    return { req: ctx.req, res: ctx.res };
  }
}
```

#### 配置

以下选项对于传递给`ThrottlerModule`数组选项的对象是有效的：

<table>
  <tr>
    <td><code>name</code></td>
    <td>用于内部跟踪正在使用的哪个节流器设置。如果没有传递，默认为`default`</td>
  </tr>
  <tr>
    <td><code>ttl</code></td>
    <td>每个请求在存储中持续的时间（以毫秒为单位）</td>
  </tr>
  <tr>
    <td><code>limit</code></td>
    <td>在TTL限制内的最大请求次数</td>
  </tr>
  <tr>
    <td><code>blockDuration</code></td>
    <td>请求被阻止的时间（以毫秒为单位）</td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>一个正则表达式数组，用于忽略节流请求时的用户代理</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>一个函数，接受<code>ExecutionContext</code>并返回一个<code>boolean</code>以短路节流器逻辑。类似于<code>@SkipThrottler()</code>，但基于请求</td>
  </tr>
</table>

如果你需要设置存储，或者想要使用上述选项中的一些以更全局的方式，适用于每个节流器设置，你可以通过`throttlers`选项键传递上述选项，并使用以下表格

<table>
  <tr>
    <td><code>storage</code></td>
    <td>自定义存储服务，用于跟踪节流行为。<a href="/security/rate-limiting#storages">查看这里。</a></td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>一个正则表达式数组，用于忽略节流请求时的用户代理</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>一个函数，接受<code>ExecutionContext</code>并返回一个<code>boolean</code>以短路节流器逻辑。类似于<code>@SkipThrottler()</code>，但基于请求</td>
  </tr>
  <tr>
    <td><code>throttlers</code></td>
    <td>一个节流器设置数组，使用上面的表格定义</td>
  </tr>
  <tr>
    <td><code>errorMessage</code></td>
    <td>一个<code>string</code>或一个函数，接受<code>ExecutionContext</code>和<code>ThrottlerLimitDetail</code>并返回一个<code>string</code>，覆盖默认的节流器错误消息</td>
  </tr>
  <tr>
    <td><code>getTracker</code></td>
    <td>一个函数，接受<code>Request</code>并返回一个<code>string</code>以覆盖默认的<code>getTracker</code>方法逻辑</td>
  </tr>
  <tr>
    <td><code>generateKey</code></td>
    <td>一个函数，接受<code>ExecutionContext</code>，跟踪器<code>string</code>和节流器名称作为<code>string</code>，并返回一个<code>string</code>以覆盖最终用于存储速率限制值的键。这覆盖了<code>generateKey</code>方法的默认逻辑</td>
  </tr>
</table>

#### 异步配置

你可能希望异步而不是同步地获取你的速率限制配置。你可以使用`forRootAsync()`方法，它允许依赖注入和`async`方法。

一种方法是使用工厂函数：

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => [
        {
          ttl: config.get('THROTTLE_TTL'),
          limit: config.get('THROTTLE_LIMIT'),
        },
      ],
    }),
  ],
})
export class AppModule {}
```

你也可以使用`useClass`语法：

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      useClass: ThrottlerConfigService,
    }),
  ],
})
export class AppModule {}
```

这是可行的，只要`ThrottlerConfigService`实现了`ThrottlerOptionsFactory`接口。

#### 存储

内置存储是内存中的缓存，它跟踪直到它们通过全局选项设置的TTL的请求。只要类实现了`ThrottlerStorage`接口，你可以将自己的存储选项放入`ThrottlerModule`的`storage`选项中。

对于分布式服务器，你可以使用社区存储提供程序[Redis](https://github.com/jmcdo29/nest-lab/tree/main/packages/throttler-storage-redis)作为单一真相来源。

> 信息提示：`ThrottlerStorage`可以从`@nestjs/throttler`导入。

#### 时间助手

有几个助手方法可以使时间更易于阅读，如果你更喜欢使用它们而不是直接定义。`@nestjs/throttler`导出了五个不同的助手，`seconds`、`minutes`、`hours`、`days`和`weeks`。使用它们，只需调用`seconds(5)`或任何其他助手，就会返回正确的毫秒数。

#### 迁移指南

对大多数人来说，将你的选项包装在一个数组中就足够了。

如果你使用自定义存储，你应该将你的`ttl`和`limit`包装在一个数组中，并分配给选项对象的`throttlers`属性。

任何`@ThrottleSkip()`现在应该接受一个对象，其`string: boolean`属性。

字符串是节流器的名称。如果没有名称，请传递字符串`'default'`，因为这将在内部使用。

任何`@Throttle()`装饰器现在也应该接受一个对象，其字符串键与节流器上下文的名称相关（同样，如果没有名称则为`'default'`），值是具有`limit`和`ttl`键的对象。

> 警告**重要**`ttl`现在是**毫秒**。如果你想保持你的ttl以秒为单位以提高可读性，请使用此包中的`seconds`助手。它只是将ttl乘以1000，使其以毫秒为单位。

更多信息，请查看[更新日志](https://github.com/nestjs/throttler/blob/master/CHANGELOG.md#500)。