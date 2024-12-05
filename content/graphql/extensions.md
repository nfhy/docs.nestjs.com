### 扩展

> 警告 **警告** 本章节仅适用于代码优先方法。

扩展是一个**高级、低级特性**，它允许你在类型配置中定义任意数据。将自定义元数据附加到特定字段，可以让你创建更复杂、更通用的解决方案。例如，通过扩展，你可以定义访问特定字段所需的字段级角色。这样的角色可以在运行时反映出来，以确定调用者是否有足够的权限来检索特定字段。

#### 添加自定义元数据

要为字段附加自定义元数据，请使用 `@nestjs/graphql` 包导出的 `@Extensions()` 装饰器。

```typescript
@Field()
@Extensions({ role: Role.ADMIN })
password: string;
```

在上面的例子中，我们将 `role` 元数据属性分配了 `Role.ADMIN` 的值。`Role` 是一个简单的 TypeScript 枚举，它聚合了我们系统中所有可用的用户角色。

请注意，除了在字段上设置元数据外，你可以在类级别和方法级别（例如，在查询处理器上）使用 `@Extensions()` 装饰器。

#### 使用自定义元数据

利用自定义元数据的逻辑可以复杂到需要的程度。例如，你可以创建一个简单的拦截器，用于存储/记录每个方法调用的事件，或者是一个[字段中间件](/graphql/field-middleware)，它匹配访问字段所需的角色与调用者权限（字段级权限系统）。

为了说明，我们定义一个 `checkRoleMiddleware`，它比较用户的角色（这里硬编码）与访问目标字段所需的角色：

```typescript
export const checkRoleMiddleware: FieldMiddleware = async (
  ctx: MiddlewareContext,
  next: NextFn,
) => {
  const { info } = ctx;
  const { extensions } = info.parentType.getFields()[info.fieldName];

  /**
   * 在真实应用中，"userRole" 变量
   * 应该代表调用者（用户）的角色（例如，"ctx.user.role"）。
   */
  const userRole = Role.USER;
  if (userRole === extensions.role) {
    // 或者简单地 "return null" 来忽略
    throw new ForbiddenException(
      `用户没有足够的权限访问 "${info.fieldName}" 字段。`,
    );
  }
  return next();
};
```

有了这个，我们可以为 `password` 字段注册一个中间件，如下所示：

```typescript
@Field({ middleware: [checkRoleMiddleware] })
@Extensions({ role: Role.ADMIN })
password: string;
```