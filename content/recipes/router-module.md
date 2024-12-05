### 路由器模块

> 信息 **提示** 本章节仅与基于HTTP的应用程序相关。

在HTTP应用程序（例如，REST API）中，处理器的路由路径是通过连接控制器声明的（可选）前缀（在 `@Controller` 装饰器内部）和方法装饰器中指定的任何路径（例如，`@Get('users')`）来确定的。你可以在[这个部分](/controllers#routing)了解更多相关信息。此外，你可以为应用程序中注册的所有路由定义一个[全局前缀](/faq/global-prefix)，或启用[版本控制](/techniques/versioning)。

同样，当定义模块级别的前缀（因此对于模块内注册的所有控制器）可能会很有用。例如，想象一个REST应用程序，它公开了几个不同的端点，这些端点被应用程序的一个特定部分称为“Dashboard”所使用。在这种情况下，你不需要在每个控制器中重复 `/dashboard` 前缀，而是可以使用一个实用程序 `RouterModule` 模块，如下所示：

```typescript
@Module({
  imports: [
    DashboardModule,
    RouterModule.register([
      {
        path: 'dashboard',
        module: DashboardModule,
      },
    ]),
  ],
})
export class AppModule {}
```

> 信息 **提示** `RouterModule` 类是从 `@nestjs/core` 包中导出的。

此外，你可以定义层次结构。这意味着每个模块都可以有 `children` 模块。子模块将继承其父模块的前缀。在以下示例中，我们将 `AdminModule` 注册为 `DashboardModule` 和 `MetricsModule` 的父模块。

```typescript
@Module({
  imports: [
    AdminModule,
    DashboardModule,
    MetricsModule,
    RouterModule.register([
      {
        path: 'admin',
        module: AdminModule,
        children: [
          {
            path: 'dashboard',
            module: DashboardModule,
          },
          {
            path: 'metrics',
            module: MetricsModule,
          },
        ],
      },
    ])
  ],
});
```

> 信息 **提示** 这个特性应该非常谨慎地使用，因为过度使用它会使代码难以维护。 

在上面的例子中，任何在 `DashboardModule` 注册的控制器都会有一个额外的 `/admin/dashboard` 前缀（因为模块会从上到下递归连接路径 - 从父模块到子模块）。同样，每个在 `MetricsModule` 定义的控制器都会有一个额外的模块级前缀 `/admin/metrics`。