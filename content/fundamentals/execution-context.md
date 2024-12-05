### 执行上下文

Nest 提供了几个实用工具类，帮助您轻松编写在多个应用程序上下文（例如，Nest HTTP 服务器基础、微服务和 WebSockets 应用程序上下文）中运行的应用程序。这些实用工具提供了关于当前执行上下文的信息，可以用来构建通用的[守卫](/guards)、[过滤器](/exception-filters)和[拦截器](/interceptors)，它们可以在广泛的控制器、方法和执行上下文中工作。

本章我们介绍两个这样的类：`ArgumentsHost` 和 `ExecutionContext`。

#### `ArgumentsHost` 类

`ArgumentsHost` 类提供了检索传递给处理器的参数的方法。它允许选择适当的上下文（例如，HTTP、RPC（微服务）或 WebSockets）来检索参数。框架在您可能想要访问它的地方提供了 `ArgumentsHost` 的一个实例，通常引用为 `host` 参数。例如，异常过滤器的 `catch()` 方法被调用时会传入一个 `ArgumentsHost` 实例。

`ArgumentsHost` 简单地作为处理器参数的抽象。例如，在 HTTP 服务器应用程序中（当使用 `@nestjs/platform-express` 时），`host` 对象封装了 Express 的 `[request, response, next]` 数组，其中 `request` 是请求对象，`response` 是响应对象，`next` 是控制应用程序请求-响应循环的函数。另一方面，在 [GraphQL](/graphql/quick-start) 应用程序中，`host` 对象包含 `[root, args, context, info]` 数组。

#### 当前应用程序上下文

当构建旨在跨多个应用程序上下文运行的通用[守卫](/guards)、[过滤器](/exception-filters)和[拦截器](/interceptors)时，我们需要一种方法来确定我们的方法当前正在运行的应用程序类型。使用 `ArgumentsHost` 的 `getType()` 方法来实现这一点：

```typescript
if (host.getType() === 'http') {
  // 执行仅在常规 HTTP 请求（REST）上下文中重要的操作
} else if (host.getType() === 'rpc') {
  // 执行仅在微服务请求上下文中重要的操作
} else if (host.getType<GqlContextType>() === 'graphql') {
  // 执行仅在 GraphQL 请求上下文中重要的操作
}
```

> **提示** `GqlContextType` 从 `@nestjs/graphql` 包中导入。

有了应用程序类型，我们可以编写更通用的组件，如下所示。

#### 主机处理器参数

要检索传递给处理器的参数数组，一种方法是使用主机对象的 `getArgs()` 方法。

```typescript
const [req, res, next] = host.getArgs();
```

您可以通过索引使用 `getArgByIndex()` 方法来获取特定的参数。

```typescript
const request = host.getArgByIndex(0);
const response = host.getArgByIndex(1);
```

在这些示例中，我们通过索引检索了请求和响应对象，这通常不推荐，因为它将应用程序与特定的执行上下文耦合。相反，您可以使用 `host` 对象的实用方法之一切换到适当的应用程序上下文，使您的代码更加健壮和可重用。上下文切换实用方法如下所示。

```typescript
/**
 * 切换上下文到 RPC。
 */
switchToRpc(): RpcArgumentsHost;
/**
 * 切换上下文到 HTTP。
 */
switchToHttp(): HttpArgumentsHost;
/**
 * 切换上下文到 WebSockets。
 */
switchToWs(): WsArgumentsHost;
```

让我们使用 `switchToHttp()` 方法重写前面的示例。`host.switchToHttp()` 辅助调用返回一个适用于 HTTP 应用程序上下文的 `HttpArgumentsHost` 对象。`HttpArgumentsHost` 对象有两个有用的方法，我们可以使用它们来提取所需的对象。在这种情况下，我们还使用 Express 类型断言来返回本地 Express 类型化对象：

```typescript
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

类似地，`WsArgumentsHost` 和 `RpcArgumentsHost` 有方法返回微服务和 WebSockets 上下文中适当的对象。以下是 `WsArgumentsHost` 的方法：

```typescript
export interface WsArgumentsHost {
  /**
   * 返回数据对象。
   */
  getData<T>(): T;
  /**
   * 返回客户端对象。
   */
  getClient<T>(): T;
}
```

以下是 `RpcArgumentsHost` 的方法：

```typescript
export interface RpcArgumentsHost {
  /**
   * 返回数据对象。
   */
  getData<T>(): T;

  /**
   * 返回上下文对象。
   */
  getContext<T>(): T;
}
```

#### `ExecutionContext` 类

`ExecutionContext` 扩展了 `ArgumentsHost`，提供了关于当前执行过程的额外详细信息。像 `ArgumentsHost` 一样，Nest 在您可能需要它的地方提供了 `ExecutionContext` 的一个实例，例如，在 [守卫](https://docs.nestjs.com/guards#execution-context) 的 `canActivate()` 方法和 [拦截器](https://docs.nestjs.com/interceptors#execution-context) 的 `intercept()` 方法中。它提供了以下方法：

```typescript
export interface ExecutionContext extends ArgumentsHost {
  /**
   * 返回当前处理器所属的控制器类的类型。
   */
  getClass<T>(): Type<T>;
  /**
   * 返回将在请求管道中下一个被调用的处理程序（方法）的引用。
   */
  getHandler(): Function;
}
```

`getHandler()` 方法返回即将被调用的处理程序的引用。`getClass()` 方法返回这个特定处理程序所属的 `Controller` 类的类型。例如，在 HTTP 上下文中，如果当前处理的请求是一个绑定到 `CatsController` 上 `create()` 方法的 `POST` 请求，`getHandler()` 返回对 `create()` 方法的引用，`getClass()` 返回 `CatsController` **类**（而不是实例）。

```typescript
const methodKey = ctx.getHandler().name; // "create"
const className = ctx.getClass().name; // "CatsController"
```

能够访问当前类和处理方法的引用提供了极大的灵活性。最重要的是，它使我们有机会访问通过 `Reflector#createDecorator` 创建的装饰器或内置的 `@SetMetadata()` 装饰器设置的元数据。我们在下面介绍这个用例。

#### 反射和元数据

Nest 提供了通过 `Reflector#createDecorator` 方法创建的装饰器和内置的 `@SetMetadata()` 装饰器将 **自定义元数据** 附加到路由处理程序的能力。在本节中，我们比较这两种方法，并看看如何在守卫或拦截器中访问元数据。

要使用 `Reflector#createDecorator` 创建强类型装饰器，我们需要指定类型参数。例如，让我们创建一个接受字符串数组作为参数的 `Roles` 装饰器。

```ts
@@filename(roles.decorator)
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

这里的 `Roles` 装饰器是一个函数，接受单个类型为 `string[]` 的参数。

现在，要使用这个装饰器，我们只需用它来注释处理程序：

```typescript
@@filename(cats.controller)
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

在这里，我们将 `Roles` 装饰器元数据附加到 `create()` 方法上，表示只有具有 `admin` 角色的用户才允许访问此路由。

要访问路由的角色（自定义元数据），我们将再次使用 `Reflector` 辅助类。`Reflector` 可以像正常一样注入到类中：

```typescript
@@filename(roles.guard)
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
@@switch
@Injectable()
@Dependencies(Reflector)
export class CatsService {
  constructor(reflector) {
    this.reflector = reflector;
  }
}
```

> **提示** `Reflector` 类从 `@nestjs/core` 包中导入。

现在，要读取处理程序元数据，使用 `get()` 方法：

```typescript
const roles = this.reflector.get(Roles, context.getHandler());
```

`Reflector#get` 方法允许我们通过传递两个参数轻松访问元数据：一个装饰器引用和一个 **上下文**（装饰器目标）来检索元数据。在这个例子中，指定的 **装饰器** 是 `Roles`（参考上面的 `roles.decorator.ts` 文件）。上下文由 `context.getHandler()` 调用提供，这导致提取当前处理的路由处理程序的元数据。记住，`getHandler()` 给我们一个 **引用** 到路由处理函数。

或者，我们可以按控制器级别组织我们的控制器，将元数据应用于控制器类的所有路由。

```typescript
@@filename(cats.controller)
@Roles(['admin'])
@Controller('cats')
export class CatsController {}
@@switch
@Roles(['admin'])
@Controller('cats')
export class CatsController {}
```

在这种情况下，要提取控制器元数据，我们传递 `context.getClass()` 作为第二个参数（提供控制器类作为元数据提取的上下文）而不是 `context.getHandler()`：

```typescript
@@filename(roles.guard)
const roles = this.reflector.get(Roles, context.getClass());
```

鉴于能够在多个级别提供元数据，您可能需要提取并合并来自多个上下文的元数据。`Reflector` 类提供了两个实用方法，用于帮助完成这项工作。这些方法一次提取 **控制器和方法** 的元数据，并以不同的方式组合它们。

考虑以下场景，您在两个级别都提供了 `Roles` 元数据。

```typescript
@@filename(cats.controller)
@Roles(['user'])
@Controller('cats')
export class CatsController {
  @Post()
  @Roles(['admin'])
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }
}
@@switch
@Roles(['user'])
@Controller('cats')
export class CatsController {}
  @Post()
  @Roles(['admin'])
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }
}
```

如果您的意图是指定 `'user'` 作为默认角色，并为某些方法选择性覆盖它，您可能会使用 `getAllAndOverride()` 方法。

```typescript
const roles = this.reflector.getAllAndOverride(Roles, [context.getHandler(), context.getClass()]);
```

在这个代码的上下文中运行的守卫，对于上述元数据，将导致 `roles` 包含 `['admin']`。

要获取元数据并合并它（这种方法合并了数组和对象），使用 `getAllAndMerge()` 方法：

```typescript
const roles = this.reflector.getAllAndMerge(Roles, [context.getHandler(), context.getClass()]);
```

这将导致 `roles` 包含 `['user', 'admin']`。

对于这两种合并方法，您将元数据键作为第一个参数传递，并将元数据目标上下文的数组（即对 `getHandler()` 和/或 `getClass()` 方法的调用）作为第二个参数传递。

#### 低级方法

如前所述，而不是使用 `Reflector#createDecorator`，您也可以使用内置的 `@SetMetadata()` 装饰器将元数据附加到处理程序。

```typescript
@@filename(cats.controller)
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@SetMetadata('roles', ['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> **提示** `@SetMetadata()` 装饰器从 `@nestjs/common` 包中导入。

通过上述构造，我们将 `roles` 元数据（`roles` 是元数据键，`['admin']` 是关联的值）附加到 `create()` 方法。虽然这有效，但直接在您的路由中使用 `@SetMetadata()` 不是一个好的实践。相反，您可以创建自己的装饰器，如下所示：

```typescript
@@filename(roles.decorator)
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
@@switch
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles) => SetMetadata('roles', roles);
```

这种方法更干净、更易读，并且在某种程度上类似于 `Reflector#createDecorator` 方法。不同之处在于，使用 `@SetMetadata` 您可以更多地控制元数据键和值，并且还可以创建接受多个参数的装饰器。

现在我们已经有一个自定义的 `@Roles()` 装饰器，我们可以使用它来装饰 `create()` 方法。

```typescript
@@filename(cats.controller)
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles('admin')
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

要访问路由的角色（自定义元数据），我们将再次使用 `Reflector` 辅助类：

```typescript
@@filename(roles.guard)
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
@@switch
@Injectable()
@Dependencies(Reflector)
export class CatsService {
  constructor(reflector) {
    this.reflector = reflector;
  }
}
```

> **提示** `Reflector` 类从 `@nestjs/core` 包中导入。

现在，要读取处理程序元数据，使用 `get()` 方法。

```typescript
const roles = this.reflector.get<string[]>('roles', context.getHandler());
```

这里，我们不是传递装饰器引用，而是将元数据 **键** 作为第一个参数传递（在我们的例子中是 `'roles'`）。其他一切都与 `Reflector#createDecorator` 示例相同。