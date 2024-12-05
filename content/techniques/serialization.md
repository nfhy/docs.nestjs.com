### 序列化

序列化是在对象返回网络响应之前发生的过程。这是提供转换和清理要返回给客户端的数据的规则的合适地方。例如，密码等敏感数据应该始终从响应中排除。或者，某些属性可能需要额外的转换，例如只发送实体的子集属性。手动执行这些转换可能会很繁琐且容易出错，并且可能会让你不确定是否已经覆盖了所有情况。

#### 概览

Nest 提供了内置功能，以帮助确保这些操作可以以直接的方式执行。`ClassSerializerInterceptor` 拦截器使用功能强大的 [class-transformer](https://github.com/typestack/class-transformer) 包来提供一种声明式和可扩展的方式来转换对象。它执行的基本操作是取方法处理器返回的值，并应用 [class-transformer](https://github.com/typestack/class-transformer) 中的 `instanceToPlain()` 函数。这样做，它可以应用在实体/DTO类上用 `class-transformer` 装饰器表达的规则，如下所述。

> 信息提示：序列化不适用于 [StreamableFile](https://docs.nestjs.com/techniques/streaming-files#streamable-file-class) 响应。

#### 排除属性

假设我们想要自动从用户实体中排除 `password` 属性。我们如下注解实体：

```typescript
import { Exclude } from 'class-transformer';

export class UserEntity {
  id: number;
  firstName: string;
  lastName: string;

  @Exclude()
  password: string;

  constructor(partial: Partial<UserEntity>) {
    Object.assign(this, partial);
  }
}
```

现在考虑一个控制器，其中的方法处理器返回这个类的实例。

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@Get()
findOne(): UserEntity {
  return new UserEntity({
    id: 1,
    firstName: 'John',
    lastName: 'Doe',
    password: 'password',
  });
}
```

> 警告：请注意，我们必须返回类的实例。如果你返回一个纯 JavaScript 对象，例如 `{ user: new UserEntity() }`，对象将不会被正确序列化。

> 信息提示：`ClassSerializerInterceptor` 从 `@nestjs/common` 导入。

当这个端点被请求时，客户端收到以下响应：

```json
{
  "id": 1,
  "firstName": "John",
  "lastName": "Doe"
}
```

请注意，拦截器可以应用到整个应用程序（如[这里](https://docs.nestjs.com/interceptors#binding-interceptors)所述）。拦截器和实体类声明的组合确保了 **任何** 返回 `UserEntity` 的方法都会确保移除 `password` 属性。这为你提供了一种集中执行此业务规则的措施。

#### 暴露属性

你可以使用 `@Expose()` 装饰器为属性提供别名，或者执行一个函数来计算属性值（类似于 **getter** 函数），如下所示。

```typescript
@Expose()
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

#### 转换

你可以使用 `@Transform()` 装饰器执行额外的数据转换。例如，以下结构返回 `RoleEntity` 的名称属性而不是返回整个对象。

```typescript
@Transform(({ value }) => value.name)
role: RoleEntity;
```

#### 传递选项

你可能想要修改转换函数的默认行为。要覆盖默认设置，请在 `options` 对象中传递它们，并使用 `@SerializeOptions()` 装饰器。

```typescript
@SerializeOptions({
  excludePrefixes: ['_'],
})
@Get()
findOne(): UserEntity {
  return new UserEntity();
}
```

> 信息提示：`@SerializeOptions()` 装饰器从 `@nestjs/common` 导入。

通过 `@SerializeOptions()` 传递的选项作为第二个参数传递给底层的 `instanceToPlain()` 函数。在这个例子中，我们自动排除所有以 `_` 前缀开头的属性。

#### 转换纯对象

你可以通过在控制器级别使用 `@SerializeOptions` 装饰器来强制执行转换。这确保了所有响应都被转换为指定类的实例，应用了来自 class-validator 或 class-transformer 的任何装饰器，即使返回的是纯对象。这种方法导致代码更干净，无需重复实例化类或调用 `plainToInstance`。

在下面的示例中，尽管在两个条件分支中都返回了纯 JavaScript 对象，但它们将自动转换为 `UserEntity` 实例，并应用相关的装饰器：

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@SerializeOptions({ type: UserEntity })
@Get()
findOne(@Query() { id }: { id: number }): UserEntity {
  if (id === 1) {
    return {
      id: 1,
      firstName: 'John',
      lastName: 'Doe',
      password: 'password',
    };
  }

  return {
    id: 2,
    firstName: 'Kamil',
    lastName: 'Mysliwiec',
    password: 'password2',
  };
}
```

> 信息提示：通过指定控制器的预期返回类型，你可以利用 TypeScript 的类型检查功能，确保返回的纯对象符合 DTO 或实体的形状。`plainToInstance` 函数不提供这种类型的提示，如果纯对象与预期的 DTO 或实体结构不匹配，可能会导致潜在的错误。

#### 示例

一个工作示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/21-serializer) 找到。

#### WebSockets 和微服务

虽然本章展示了使用 HTTP 风格应用程序（例如，Express 或 Fastify）的示例，`ClassSerializerInterceptor` 对于 WebSockets 和微服务的工作方式相同，无论使用哪种传输方法。

#### 了解更多

了解更多由 `class-transformer` 包提供的装饰器和选项，请访问 [这里](https://github.com/typestack/class-transformer)。