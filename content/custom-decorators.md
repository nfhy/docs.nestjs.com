### 自定义路由装饰器

Nest 构建于一个称为 **装饰器** 的语言特性之上。装饰器在许多常用的编程语言中是一个众所周知的概念，但在 JavaScript 领域，它们仍然是相对较新的。为了更好地理解装饰器的工作原理，我们推荐阅读[这篇文章](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841)。这里有一个简单的定义：

<blockquote class="external">
  ES2016 装饰器是一个表达式，它返回一个函数，并且可以接受目标、名称和属性描述符作为参数。
  你通过在装饰器前加上 <code>@</code> 字符，并将此放在你想要装饰的内容的最顶部来应用它。
  装饰器可以定义在类、方法或属性上。
</blockquote>

#### 参数装饰器

Nest 提供了一系列有用的 **参数装饰器**，你可以将它们与 HTTP 路由处理器一起使用。以下是提供的装饰器列表以及它们代表的纯 Express（或 Fastify）对象：

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td>
    </tr>
    <tr>
      <td><code>@Response(), @Res()</code></td>
      <td><code>res</code></td>
    </tr>
    <tr>
      <td><code>@Next()</code></td>
      <td><code>next</code></td>
    </tr>
    <tr>
      <td><code>@Session()</code></td>
      <td><code>req.session</code></td>
    </tr>
    <tr>
      <td><code>@Param(param?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[param]</code></td>
    </tr>
    <tr>
      <td><code>@Body(param?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[param]</code></td>
    </tr>
    <tr>
      <td><code>@Query(param?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[param]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(param?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[param]</code></td>
    </tr>
    <tr>
      <td><code>@Ip()</code></td>
      <td><code>req.ip</code></td>
    </tr>
    <tr>
      <td><code>@HostParam()</code></td>
      <td><code>req.hosts</code></td>
    </tr>
  </tbody>
</table>

此外，你可以创建自己的 **自定义装饰器**。这有什么用呢？

在 node.js 世界中，常见的做法是将属性附加到 **请求** 对象上。然后你在每个路由处理器中手动提取它们，使用如下代码：

```typescript
const user = req.user;
```

为了使你的代码更易于阅读和透明，你可以创建一个 `@User()` 装饰器，并在所有控制器中重用它。

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

然后，你可以在任何需要的地方简单地使用它。

```typescript
@@filename()
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
@@switch
@Get()
@Bind(User())
async findOne(user) {
  console.log(user);
}
```

#### 传递数据

当你的装饰器行为取决于某些条件时，你可以使用 `data` 参数向装饰器的工厂函数传递一个参数。一个用例是自定义装饰器，它通过键从请求对象中提取属性。假设，例如，我们的 <a href="techniques/authentication#implementing-passport-strategies">认证层</a> 验证请求并将用户实体附加到请求对象上。经过身份验证的请求的用户实体可能如下所示：

```json
{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

让我们定义一个装饰器，它接受属性名称作为键，并在存在时返回关联的值（如果不存在，则返回 undefined，或者如果 `user` 对象尚未创建）。

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
@@switch
import { createParamDecorator } from '@nestjs/common';

export const User = createParamDecorator((data, ctx) => {
  const request = ctx.switchToHttp().getRequest();
  const user = request.user;

  return data ? user && user[data] : user;
});
```

以下是你如何在控制器中通过 `@User()` 装饰器访问特定属性：

```typescript
@@filename()
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
@@switch
@Get()
@Bind(User('firstName'))
async findOne(firstName) {
  console.log(`Hello ${firstName}`);
}
```

你可以使用这个相同的装饰器与不同的键来访问不同的属性。如果 `user` 对象是深层或复杂的，这可以使请求处理器实现更简单、更易于阅读。

> 提示：对于 TypeScript 用户，请注意 `createParamDecorator<T>()` 是一个泛型。这意味着你可以显式地强制类型安全，例如 `createParamDecorator<string>((data, ctx) => ...)`。或者，在工厂函数中指定参数类型，例如 `createParamDecorator((data: string, ctx) => ...)`。如果你两者都省略，`data` 的类型将是 `any`。

#### 与管道一起工作

Nest 对自定义参数装饰器的处理方式与内置装饰器（`@Body()`, `@Param()` 和 `@Query()`）相同。这意味着管道也会为自定义注解参数执行（在我们的例子中，`user` 参数）。此外，你可以直接将管道应用于自定义装饰器：

```typescript
@@filename()
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
@@switch
@Get()
@Bind(User(new ValidationPipe({ validateCustomDecorators: true })))
async findOne(user) {
  console.log(user);
}
```

> 提示：请注意 `validateCustomDecorators` 选项必须设置为 true。`ValidationPipe` 默认不验证用自定义装饰器注解的参数。

#### 装饰器组合

Nest 提供了一个辅助方法来组合多个装饰器。例如，假设你想将所有与认证相关的装饰器组合成一个装饰器。这可以通过以下构造完成：

```typescript
@@filename(auth.decorator)
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
@@switch
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

然后你可以如下使用这个自定义的 `@Auth()` 装饰器：

```typescript
@Get('users')
@Auth('admin')
findAllUsers() {}
```

这相当于用单个声明应用所有四个装饰器。

> 警告：`@nestjs/swagger` 包中的 `@ApiHideProperty()` 装饰器不可组合，并且使用 `applyDecorators` 函数时不会正常工作。