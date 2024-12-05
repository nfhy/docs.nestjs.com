### 独立应用程序

有几种方式可以挂载Nest应用程序。您可以创建一个Web应用程序、一个微服务或者只是一个没有网络监听器的Nest**独立应用程序**。Nest独立应用程序是Nest**IoC容器**的包装器，它包含所有实例化的类。我们可以直接使用独立应用程序对象从任何导入的模块中获取任何现有实例的引用。因此，您可以在任何地方利用Nest框架，包括例如脚本化的**CRON**作业。您甚至可以在其上构建一个**CLI**。

#### 开始使用

要创建一个Nest独立应用程序，请使用以下结构：

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // 您的应用程序逻辑在这里...
}
bootstrap();
```

#### 从静态模块检索提供者

独立应用程序对象允许您获取在Nest应用程序中注册的任何实例的引用。假设我们在`AppModule`模块导入的`TasksModule`模块中有一个`TasksService`提供者，这个类提供了我们想要在CRON作业中调用的一组方法。

```typescript
@@filename()
const tasksService = app.get(TasksService);
```

要访问`TasksService`实例，我们使用`get()`方法。`get()`方法像一个**查询**，搜索每个注册模块中的实例。您可以向它传递任何提供者的令牌。或者，对于严格的上下文检查，传递一个带有`strict: true`属性的选项对象。启用此选项后，您必须通过特定模块导航以从选定的上下文中获取特定实例。

```typescript
@@filename()
const tasksService = app.select(TasksModule).get(TasksService, { strict: true });
```

以下是可用于从独立应用程序对象检索实例引用的方法的摘要。

<table>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      检索在应用程序上下文中可用的控制器或提供者（包括守卫、过滤器等）的实例。
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      遍历模块图以提取选定模块的特定实例（如上所述，与严格模式一起使用）。
    </td>
  </tr>
</table>

> 信息 **提示** 在非严格模式下，默认选择根模块。要选择任何其他模块，您需要手动逐步遍历模块图。

请记住，独立应用程序没有任何网络监听器，因此任何与HTTP相关的Nest功能（例如，中间件、拦截器、管道、守卫等）在此上下文中不可用。

例如，即使您在应用程序中注册了一个全局拦截器，然后使用`app.get()`方法检索控制器的实例，拦截器也不会被执行。

#### 从动态模块检索提供者

在处理[动态模块](./fundamentals/dynamic-modules.md)时，我们应该向`app.select`提供相同的对象，该对象代表应用程序中注册的动态模块。例如：

```typescript
@@filename()
export const dynamicConfigModule = ConfigModule.register({ folder: './config' });

@Module({
  imports: [dynamicConfigModule],
})
export class AppModule {}
```

然后您可以稍后选择该模块：

```typescript
@@filename()
const configService = app.select(dynamicConfigModule).get(ConfigService, { strict: true });
```

#### 终止阶段

如果您希望在脚本完成后关闭Node应用程序（例如，对于运行CRON作业的脚本），您必须在`bootstrap`函数的末尾调用`app.close()`方法，如下所示：

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // 应用程序逻辑...
  await app.close();
}
bootstrap();
```

如在[生命周期事件](./fundamentals/lifecycle-events.md)章节中提到的，这将触发生命周期钩子。

#### 示例

一个工作示例可在[这里](https://github.com/nestjs/nest/tree/master/sample/18-context)找到。