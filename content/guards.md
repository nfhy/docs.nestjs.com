### 守卫

守卫是一个用 `@Injectable()` 装饰器注解的类，它实现了 `CanActivate` 接口。

<figure><img class="illustrative-image" src="/assets/Guards_1.png" /></figure>

守卫有**单一职责**。它们根据运行时的条件（如权限、角色、访问控制列表等）来决定是否将给定的请求交给路由处理器处理。这通常被称为**授权**。授权（及其表亲**认证**，通常与之合作）通常由传统 Express 应用程序中的[中间件](/middleware)处理。中间件是认证的好选择，因为像令牌验证和附加属性到 `request` 对象这样的操作与特定路由上下文（及其元数据）没有强关联。

但中间件，就其本质而言，是愚蠢的。它不知道在调用 `next()` 函数后将执行哪个处理器。另一方面，**守卫**可以访问 `ExecutionContext` 实例，因此确切知道接下来将执行什么。它们被设计成，就像异常过滤器、管道和拦截器一样，允许你在请求/响应周期的确切正确点插入处理逻辑，并且以声明性的方式进行。这有助于保持你的代码 DRY 和声明性。

> 信息提示：守卫在所有中间件之后执行，但在任何拦截器或管道之前。

#### 授权守卫

如上所述，**授权**是守卫的一个绝佳用例，因为特定路由应该只在呼叫者（通常是特定经过认证的用户）具有足够权限时才可用。我们现在要构建的 `AuthGuard` 假设有一个经过认证的用户（因此，令牌附加在请求头中）。它将提取并验证令牌，并使用提取的信息来决定请求是否可以继续。

```typescript
@@filename(auth.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class AuthGuard {
  async canActivate(context) {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

> 信息提示：如果您正在寻找如何在您的应用程序中实现认证机制的现实世界示例，请访问[本章节](/security/authentication)。同样，对于更复杂的授权示例，请查看[本页面](/security/authorization)。

`validateRequest()` 函数内的逻辑可以简单也可以复杂。这个示例的主要目的是展示守卫如何适应请求/响应周期。

每个守卫都必须实现一个 `canActivate()` 函数。这个函数应该返回一个布尔值，指示当前请求是否被允许。它可以同步或异步返回响应（通过 `Promise` 或 `Observable`）。Nest 使用返回值来控制下一个动作：

- 如果返回 `true`，则请求将被处理。
- 如果返回 `false`，则 Nest 将拒绝请求。

<app-banner-enterprise></app-banner-enterprise>

#### 执行上下文

`canActivate()` 函数接受一个参数，即 `ExecutionContext` 实例。`ExecutionContext` 继承自 `ArgumentsHost`。我们在异常过滤器章节之前看到了 `ArgumentsHost`。在上面的示例中，我们只是使用 `ArgumentsHost` 上定义的相同辅助方法来获取对 `Request` 对象的引用。您可以回顾[异常过滤器](https://docs.nestjs.com/exception-filters#arguments-host)章节中的**参数宿主**部分，了解更多关于此主题的信息。

通过扩展 `ArgumentsHost`，`ExecutionContext` 还添加了几个新的辅助方法，提供了关于当前执行过程的额外详细信息。这些详细信息可以帮助构建更通用的守卫，这些守卫可以在广泛的控制器、方法和执行上下文中工作。了解更多关于 `ExecutionContext` 的信息[在这里](/fundamentals/execution-context)。

#### 基于角色的认证

让我们构建一个更实用的守卫，只允许具有特定角色的用户访问。我们现在从一个基本的守卫模板开始，并在接下来的部分中构建。目前，它允许所有请求继续：

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class RolesGuard {
  canActivate(context) {
    return true;
  }
}
```

#### 绑定守卫

像管道和异常过滤器一样，守卫可以是**控制器作用域**的、方法作用域的或全局作用域的。下面，我们使用 `@UseGuards()` 装饰器设置了一个控制器作用域的守卫。这个装饰器可以接收单个参数，或逗号分隔的参数列表。这让您可以轻松地用一个声明应用适当的一组守卫。

```typescript
@@filename()
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> 信息提示：`@UseGuards()` 装饰器从 `@nestjs/common` 包导入。

上述，我们传递了 `RolesGuard` 类（而不是实例），将实例化的职责留给了框架，并启用了依赖注入。与管道和异常过滤器一样，我们也可以通过以下方式传递一个就地实例：

```typescript
@@filename()
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

上述构造将守卫附加到此控制器声明的每个处理器上。如果我们希望守卫只适用于单个方法，我们在**方法级别**应用 `@UseGuards()` 装饰器。

为了设置一个全局守卫，使用 Nest 应用程序实例的 `useGlobalGuards()` 方法：

```typescript
@@filename()
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> 警告：在混合应用程序的情况下，`useGlobalGuards()` 方法不会默认为网关和微服务设置守卫（有关如何更改此行为的信息，请参见[混合应用程序](/faq/hybrid-application)）。对于“标准”（非混合）微服务应用程序，`useGlobalGuards()` 确实会全局安装守卫。

全局守卫用于整个应用程序，每个控制器和每个路由处理器。在依赖注入方面，从任何模块外部注册的全局守卫（如上例中的 `useGlobalGuards()`）不能注入依赖项，因为这是在任何模块的上下文之外完成的。为了解决这个问题，您可以直接从任何模块设置守卫，如下所示：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> 信息提示：当使用这种方法为守卫执行依赖注入时，请注意，无论
> 在哪里使用这种构造的模块，守卫实际上都是全局的。应该在哪里完成？
> 选择守卫（示例中的 `RolesGuard`）定义的模块。此外，`useClass` 不是处理
> 自定义提供者注册的唯一方式。了解更多[在这里](/fundamentals/custom-providers)。

#### 为处理器设置角色

我们的 `RolesGuard` 正在工作，但它还不够智能。我们还没有利用守卫最重要的特性 - [执行上下文](/fundamentals/execution-context)。它还不知道角色，或者哪些角色允许每个处理器。例如，`CatsController` 可能对不同路由有不同的权限方案。有些可能只对管理员用户可用，其他可能对所有人开放。我们如何以灵活且可重用的方式将角色与路由匹配？

这就是 **自定义元数据** 的用武之地（了解更多[在这里](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)）。Nest 提供了通过 `Reflector#createDecorator` 静态方法创建的装饰器或内置的 `@SetMetadata()` 装饰器将自定义 **元数据** 附加到路由处理器的能力。

例如，让我们使用 `Reflector#createDecorator` 方法创建一个 `@Roles()` 装饰器，它将元数据附加到处理器。`Reflector` 由框架提供，并从 `@nestjs/core` 包中公开。

```ts
@@filename(roles.decorator)
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

`Roles` 装饰器是一个接受单个类型为 `string[]` 参数的函数。

现在，要使用这个装饰器，我们只需用它来注解处理器：

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

在这里，我们将 `Roles` 装饰器元数据附加到 `create()` 方法，表示只有具有 `admin` 角色的用户才允许访问此路由。

或者，而不是使用 `Reflector#createDecorator` 方法，我们可以使用内置的 `@SetMetadata()` 装饰器。了解更多[在这里](/fundamentals/execution-context#low-level-approach)。

#### 整合在一起

现在让我们回到我们的 `RolesGuard`。目前，它在所有情况下都简单地返回 `true`，允许每个请求继续。我们希望基于比较 **当前用户分配的角色** 与当前正在处理的路由所需的实际角色来使返回值条件化。为了访问路由的角色（自定义元数据），我们将再次使用 `Reflector` 辅助类，如下所示：

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

> 信息提示：在 node.js 世界中，将授权用户附加到 `request` 对象是一种常见做法。因此，在我们的示例代码中，我们假设 `request.user` 包含用户实例和允许的角色。在您的应用程序中，您可能会在自定义 **认证守卫**（或中间件）中进行这种关联。有关更多信息，请查看[本章节](/security/authentication)。

> 警告：`matchRoles()` 函数内的逻辑可以简单也可以复杂。这个示例的主要目的是展示守卫如何适应请求/响应周期。

参考 **执行上下文** 章节中的 [反射和元数据](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata) 部分，了解更多关于上下文敏感地使用 `Reflector` 的详细信息。

当权限不足的用户请求端点时，Nest 自动返回以下响应：

```typescript
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

请注意，在幕后，当守卫返回 `false` 时，框架会抛出一个 `ForbiddenException`。如果您想返回不同的错误响应，您应该抛出自己的特定异常。例如：

```typescript
throw new UnauthorizedException();
```

守卫抛出的任何异常都将由[异常层](/exception-filters)处理（全局异常过滤器和应用于当前上下文的任何异常过滤器）。

> 信息提示：如果您正在寻找如何实现授权的现实世界示例，请查看[本章节](/security/authorization)。