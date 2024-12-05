### Apollo 插件

插件使您能够通过在响应特定事件时执行自定义操作来扩展 Apollo Server 的核心功能。目前，这些事件对应于 GraphQL 请求生命周期的各个阶段，以及 Apollo Server 本身的启动（更多信息[点此](https://www.apollographql.com/docs/apollo-server/integrations/plugins/)）。例如，一个基本的日志记录插件可能会记录发送到 Apollo Server 的每个请求的 GraphQL 查询字符串。

#### 自定义插件

要创建一个插件，声明一个用 `@nestjs/apollo` 包中的 `@Plugin` 装饰器注解的类，并实现 `@apollo/server` 包中的 `ApolloServerPlugin` 接口以获得更好的代码自动补全。

```typescript
import { ApolloServerPlugin, GraphQLRequestListener } from '@apollo/server';
import { Plugin } from '@nestjs/apollo';

@Plugin()
export class LoggingPlugin implements ApolloServerPlugin {
  async requestDidStart(): Promise<GraphQLRequestListener<any>> {
    console.log('请求开始');
    return {
      async willSendResponse() {
        console.log('将发送响应');
      },
    };
  }
}
```

有了这个，我们可以将 `LoggingPlugin` 作为提供者注册。

```typescript
@Module({
  providers: [LoggingPlugin],
})
export class CommonModule {}
```

Nest 将自动实例化插件并将其应用于 Apollo Server。

#### 使用外部插件

提供了几个现成的插件。要使用现有的插件，只需导入它并将其添加到 `plugins` 数组中：

```typescript
GraphQLModule.forRoot({
  // ...
  plugins: [ApolloServerOperationRegistry({ /* 选项 */})]
}),
```

> 信息提示 **提示** `ApolloServerOperationRegistry` 插件是从 `@apollo/server-plugin-operation-registry` 包中导出的。

#### Mercurius 插件

一些现有的特定于 mercurius 的 Fastify 插件必须在插件树中加载在 mercurius 插件之后（更多信息[点此](https://mercurius.dev/#/docs/plugins)）。

> 警告 **警告** [mercurius-upload](https://github.com/mercurius-js/mercurius-upload) 是一个例外，应该在主文件中注册。

为此，`MercuriusDriver` 提供了一个可选的 `plugins` 配置选项。它代表一个对象数组，由两个属性组成：`plugin` 和它的 `options`。因此，注册 [缓存插件](https://github.com/mercurius-js/cache) 看起来会是这样的：

```typescript
GraphQLModule.forRoot({
  driver: MercuriusDriver,
  // ...
  plugins: [
    {
      plugin: cache,
      options: {
        ttl: 10,
        policy: {
          Query: {
            add: true
          }
        }
      },
    }
  ]
}),
```