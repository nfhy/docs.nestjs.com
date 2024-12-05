### 注入作用域

对于来自不同编程语言背景的人来说，了解到在 Nest 中几乎所有东西都是跨请求共享的可能会感到意外。我们有一个数据库的连接池，具有全局状态的单例服务等。请记住，Node.js 不遵循每个请求由单独线程处理的请求/响应多线程无状态模型。因此，在我们的应用程序中使用单例实例是完全**安全**的。

然而，在某些边缘情况下，可能需要基于请求的生命周期行为，例如 GraphQL 应用程序中的按请求缓存、请求跟踪和多租户。注入作用域提供了一种机制来获得所需的提供者生命周期行为。

#### 提供者作用域

提供者可以具有以下任何作用域：

| 提供者作用域 | 描述 |
| --- | --- |
| `DEFAULT` | 整个应用程序共享单个提供者实例。实例的生命周期直接与应用程序生命周期相关联。一旦应用程序启动引导完成，所有单例提供者都已被实例化。默认情况下使用单例作用域。 |
| `REQUEST` | 为每个传入的**请求**专门创建一个新的提供者实例。请求处理完成后，实例将被垃圾回收。 |
| `TRANSIENT` | 瞬态提供者不会在消费者之间共享。每个注入瞬态提供者的消费者将收到一个新的、专用的实例。 |

> **提示** 使用单例作用域是**推荐**的大多数用例。在消费者和请求之间共享提供者意味着可以缓存一个实例，并且其初始化仅在应用程序启动期间发生一次。

#### 使用方法

通过将 `scope` 属性传递给 `@Injectable()` 装饰器选项对象来指定注入作用域：

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

同样，对于[自定义提供者](/fundamentals/custom-providers)，在提供者注册的长格式中设置 `scope` 属性：

```typescript
{
  provide: 'CACHE_MANAGER',
  useClass: CacheManager,
  scope: Scope.TRANSIENT,
}
```

> **提示** 从 `@nestjs/common` 导入 `Scope` 枚举

默认情况下使用单例作用域，不需要声明。如果您确实想声明一个提供者为单例作用域，请使用 `Scope.DEFAULT` 值作为 `scope` 属性的值。

> **注意** Websocket 网关不应使用请求作用域提供者，因为它们必须作为单例。每个网关封装了一个真实的套接字，并且不能多次实例化。这个限制也适用于其他一些提供者，如[_Passport 策略_](../security/authentication#request-scoped-strategies)或_Cron 控制器_。

#### 控制器作用域

控制器也可以有作用域，这适用于该控制器中声明的所有请求方法处理程序。像提供者作用域一样，控制器的作用域声明了其生命周期。对于请求作用域的控制器，每个传入请求都会为其创建一个新实例，并在请求处理完成后进行垃圾回收。

使用 `ControllerOptions` 对象的 `scope` 属性声明控制器作用域：

```typescript
@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```

#### 作用域层次结构

`REQUEST` 作用域会沿着注入链冒泡。依赖于请求作用域提供者的控制器本身也将是请求作用域的。

想象一下以下依赖图：`CatsController <- CatsService <- CatsRepository`。如果 `CatsService` 是请求作用域的（而其他是默认的单例），那么 `CatsController` 将成为请求作用域的，因为它依赖于注入的服务。`CatsRepository` 不依赖于服务，将保持单例作用域。

瞬态作用域的依赖不遵循该模式。如果单例作用域的 `DogsService` 注入了瞬态 `LoggerService` 提供者，它将收到一个新鲜的实例。然而，`DogsService` 将保持单例作用域，因此，在任何地方注入它都不会解析为 `DogsService` 的新实例。如果这是期望的行为，`DogsService` 必须显式标记为 `TRANSIENT`。

#### 请求提供者

在基于 HTTP 服务器的应用程序中（例如，使用 `@nestjs/platform-express` 或 `@nestjs/platform-fastify`），您可能希望在使用请求作用域提供者时访问原始请求对象的引用。您可以通过注入 `REQUEST` 对象来实现这一点。

`REQUEST` 提供者固有地是请求作用域的，这意味着在使用它时不需要显式指定 `REQUEST` 作用域。此外，即使您尝试这样做，它也会被忽略。任何依赖于请求作用域提供者的提供者都会自动采用请求作用域，这种行为不能更改。

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

由于底层平台/协议差异，您在微服务或 GraphQL 应用程序中访问传入请求的方式略有不同。在[GraphQL](/graphql/quick-start)应用程序中，您注入 `CONTEXT` 而不是 `REQUEST`：

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT } from '@nestjs/graphql';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

然后您配置您的 `context` 值（在 `GraphQLModule` 中）以包含 `request` 作为其属性。

#### 询问者提供者

如果您想要获取提供者构造的位置的类，例如在日志记录或指标提供者中，您可以注入 `INQUIRER` 令牌。

```typescript
import { Inject, Injectable, Scope } from '@nestjs/common';
import { INQUIRER } from '@nestjs/core';

@Injectable({ scope: Scope.TRANSIENT })
export class HelloService {
  constructor(@Inject(INQUIRER) private parentClass: object) {}

  sayHello(message: string) {
    console.log(`${this.parentClass?.constructor?.name}: ${message}`);
  }
}
```

然后可以这样使用：

```typescript
import { Injectable } from '@nestjs/common';
import { HelloService } from './hello.service';

@Injectable()
export class AppService {
  constructor(private helloService: HelloService) {}

  getRoot(): string {
    this.helloService.sayHello('My name is getRoot');

    return 'Hello world!';
  }
}
```

在上面的例子中，当调用 `AppService#getRoot` 时，`"AppService: My name is getRoot"` 将被记录到控制台。

#### 性能

使用请求作用域提供者将对应用程序性能产生影响。虽然 Nest 尝试尽可能多地缓存元数据，但它仍然需要在每个请求上创建您的类的实例。因此，它将减慢您的平均响应时间和整体基准测试结果。除非提供者必须是请求作用域的，否则强烈建议您使用默认的单例作用域。

> **提示** 尽管听起来相当令人生畏，一个合理设计的应用程序，利用请求作用域提供者不应该使延迟超过 ~5%。

#### 持久提供者

如上所述的请求作用域提供者可能会导致延迟增加，因为至少有一个请求作用域提供者（注入到控制器实例中，或更深层次 - 注入到其一个提供者中）也会使控制器成为请求作用域的。这意味着它必须为每个单独的请求重新创建（实例化），并在之后进行垃圾收集。现在，这意味着，比如说有 30k 并发请求，将会有 30k 个控制器（及其请求作用域提供者）的临时实例。

如果有一个大多数提供者依赖的共同提供者（想想数据库连接，或日志记录服务），它会自动将所有这些提供者转换为请求作用域提供者。这在**多租户应用程序**中可能是一个挑战，特别是对于那些有一个中心请求作用域“数据源”提供者根据请求对象获取标头/令牌，并基于其值检索相应的数据库连接/架构（特定于该租户）的应用程序。

例如，假设您有一个应用程序，它交替被 10 个不同的客户使用。每个客户都有自己的专用数据源，您希望确保客户 A 永远无法访问客户 B 的数据库。实现这一点的一种方式是声明一个请求作用域的“数据源”提供者 - 根据请求对象确定“当前客户”并检索其相应的数据库。通过这种方法，您可以在短短几分钟内将您的应用程序转变为多租户应用程序。但是，这种方法的一个主要缺点是，由于您的应用程序的大部分组件可能依赖于“数据源”提供者，它们将隐式地成为“请求作用域”，因此您无疑会在应用程序性能上看到影响。

但是如果我们有更好的解决方案呢？由于我们只有 10 个客户，我们是否可以为每个客户拥有 10 个单独的[DI 子树](/fundamentals/module-ref#resolving-scoped-providers)（而不是为每个传入请求重新创建每个树）？如果您的提供者不依赖于任何真正独特的属性（例如，请求 UUID），而是有一些特定的属性让我们可以聚合（分类）它们，那么没有理由在每个传入请求上重新创建 DI 子树。

这正是**持久提供者**派上用场的地方。

在我们开始将提供者标记为持久之前，我们首先必须注册一个**策略**，该策略指导 Nest 什么是那些“共同请求属性”，提供逻辑，将请求分组 - 将它们与相应的 DI 子树关联起来。

```typescript
import {
  HostComponentInfo,
  ContextId,
  ContextIdFactory,
  ContextIdStrategy,
} from '@nestjs/core';
import { Request } from 'express';

const tenants = new Map<string, ContextId>();

export class AggregateByTenantContextIdStrategy implements ContextIdStrategy {
  attach(contextId: ContextId, request: Request) {
    const tenantId = request.headers['x-tenant-id'] as string;
    let tenantSubTreeId: ContextId;

    if (tenants.has(tenantId)) {
      tenantSubTreeId = tenants.get(tenantId);
    } else {
      tenantSubTreeId = ContextIdFactory.create();
      tenants.set(tenantId, tenantSubTreeId);
    }

    // 如果树不是持久的，返回原始的“contextId”对象
    return (info: HostComponentInfo) =>
      info.isTreeDurable ? tenantSubTreeId : contextId;
  }
}
```

> **提示** 类似于请求作用域，持久性会沿着注入链冒泡。这意味着如果 A 依赖于被标记为 `durable` 的 B，A 也会隐式地成为持久的（除非为 A 提供者显式设置 `durable` 为 `false`）。

> **警告** 注意这个策略不适用于拥有大量租户的应用程序。

`attach` 方法返回的值指示 Nest 应该使用哪个上下文标识符用于给定的主机。在这种情况下，我们指定应该使用 `tenantSubTreeId` 而不是原始的自动生成的 `contextId` 对象，当主机组件（例如，请求作用域控制器）被标记为持久时。此外，在上述示例中，**不注册**有效载荷（有效载荷 = `REQUEST`/`CONTEXT` 提供者，代表“根” - 子树的父级）。

如果您想要为持久树注册有效载荷，使用以下构造代替：

```typescript
// `AggregateByTenantContextIdStrategy#attach` 方法的返回：
return {
  resolve: (info: HostComponentInfo) =>
    info.isTreeDurable ? tenantSubTreeId : contextId,
  payload: { tenantId },
};
```

现在，每当您使用 `@Inject(REQUEST)`/`@Inject(CONTEXT)` 注入 `REQUEST` 提供者（或 GraphQL 应用程序中的 `CONTEXT`）时，将注入 `payload` 对象（由一个属性组成 - 本例中的 `tenantId`）。

好的，有了这个策略，您可以在代码中的任何地方注册它（因为它无论如何都适用全局），例如，您可以将其放在 `main.ts` 文件中：

```typescript
ContextIdFactory.apply(new AggregateByTenantContextIdStrategy());
```

> **提示** `ContextIdFactory` 类是从 `@nestjs/core` 包导入的。

只要注册发生在任何请求击中您的应用程序之前，一切都会按预期工作。

最后，要将常规提供者转换为持久提供者，只需将 `durable` 标志设置为 `true` 并将其作用域更改为 `Scope.REQUEST`（如果注入链中已经有 REQUEST 作用域，则不需要）：

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST, durable: true })
export class CatsService {}
```

同样，对于[自定义提供者](/fundamentals/custom-providers)，在提供者注册的长格式中设置 `durable` 属性：

```typescript
{
  provide: 'foobar',
  useFactory: () => { ... },
  scope: Scope.REQUEST,
  durable: true,
}
```