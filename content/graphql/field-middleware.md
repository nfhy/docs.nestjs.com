### 字段中间件

> 警告 **警告** 本章节仅适用于代码优先方法。

字段中间件允许您在字段解析前后运行任意代码。字段中间件可用于转换字段结果、验证字段参数，甚至检查字段级别的角色（例如，需要访问执行中间件函数的目标字段）。

您可以将多个中间件函数连接到一个字段。在这种情况下，它们将按顺序沿链被调用，前一个中间件决定调用下一个中间件。中间件函数在 `middleware` 数组中的顺序很重要。第一个解析器是最外层，因此它首先和最后执行（类似于 `graphql-middleware` 包）。第二个解析器是第二外层，因此它第二和倒数第二执行。

#### 开始使用

让我们从一个简单的中间件开始，该中间件将在字段值发送回客户端之前记录它：

```typescript
import { FieldMiddleware, MiddlewareContext, NextFn } from '@nestjs/graphql';

const loggerMiddleware: FieldMiddleware = async (
  ctx: MiddlewareContext,
  next: NextFn,
) => {
  const value = await next();
  console.log(value);
  return value;
};
```

> 提示 **提示** `MiddlewareContext` 是一个对象，包含通常由 GraphQL 解析器函数接收的相同参数（`{ source, args, context, info }`），而 `NextFn` 是一个函数，允许您执行堆栈中的下一个中间件（绑定到该字段）或实际字段解析器。

> 警告 **警告** 字段中间件函数不能注入依赖项，也不能访问 Nest 的 DI 容器，因为它们被设计为非常轻量级，不应该执行任何可能耗时的操作（比如从数据库检索数据）。如果您需要调用外部服务/从数据源查询数据，您应该在绑定到根查询/突变处理器的守卫/拦截器中执行此操作，并将其分配给 `context` 对象，您可以在字段中间件中从 `MiddlewareContext` 对象中访问它。

请注意，字段中间件必须匹配 `FieldMiddleware` 接口。在上面的例子中，我们首先运行 `next()` 函数（执行实际字段解析器并返回字段值），然后，我们将这个值记录到我们的终端。同样，从中间件函数返回的值完全覆盖了前一个值，由于我们不想执行任何更改，我们简单地返回原始值。

有了这个，我们可以直接在 `@Field()` 装饰器中注册我们的中间件，如下所示：

```typescript
@ObjectType()
export class Recipe {
  @Field({ middleware: [loggerMiddleware] })
  title: string;
}
```

现在，每当我们请求 `Recipe` 对象类型的 `title` 字段时，原始字段的值将被记录到控制台。

> 提示 **提示** 要了解如何使用 [extensions](/graphql/extensions) 功能实现字段级权限系统，请查看此 [部分](/graphql/extensions#using-custom-metadata)。

> 警告 **警告** 字段中间件只能应用于 `ObjectType` 类。更多详情，请查看此 [问题](https://github.com/nestjs/graphql/issues/2446)。

同样，如上所述，我们可以在中间件函数内控制字段的值。为了演示，让我们将食谱的标题（如果存在）大写：

```typescript
const value = await next();
return value?.toUpperCase();
```

在这种情况下，每个标题在请求时将自动大写。

同样，您可以将字段中间件绑定到自定义字段解析器（用 `@ResolveField()` 装饰器注解的方法），如下所示：

```typescript
@ResolveField(() => String, { middleware: [loggerMiddleware] })
title() {
  return 'Placeholder';
}
```

> 警告 **警告** 如果在字段解析器级别启用了增强器（[了解更多](/graphql/other-features#execute-enhancers-at-the-field-resolver-level)），字段中间件函数将在任何绑定到方法的拦截器、守卫等之前运行（但在为查询或突变处理器注册的根级增强器之后）。

#### 全局字段中间件

除了直接将中间件绑定到特定字段外，您还可以注册一个或多个全局中间件函数。在这种情况下，它们将自动连接到您的对象类型的所有字段。

```typescript
GraphQLModule.forRoot({
  autoSchemaFile: 'schema.gql',
  buildSchemaOptions: {
    fieldMiddleware: [loggerMiddleware],
  },
}),
```

> 提示 **提示** 全局注册的字段中间件函数将在本地注册的中间件函数之前执行（那些直接绑定到特定字段的）。