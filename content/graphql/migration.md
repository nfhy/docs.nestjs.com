### 从 v10 迁移到 v11

本章节提供了从 `@nestjs/graphql` 版本 10 迁移到版本 11 的一系列指南。在这个主要版本发布中，我们更新了 Apollo 驱动，使其与 Apollo Server v4（而不是 v3）兼容。注意：Apollo Server v4 中有几个重大变更（特别是在插件和生态系统包方面），因此您必须相应地更新代码库。更多信息，请参见 [Apollo Server v4 迁移指南](https://www.apollographql.com/docs/apollo-server/migration/)。

#### Apollo 包

您不再需要安装 `apollo-server-express` 包，而是需要安装 `@apollo/server`：

```bash
$ npm uninstall apollo-server-express
$ npm install @apollo/server
```

如果您使用 Fastify 适配器，则需要安装 `@as-integrations/fastify` 包：

```bash
$ npm uninstall apollo-server-fastify
$ npm install @apollo/server @as-integrations/fastify
```

#### Mercurius 包

Mercurius 网关不再是 `mercurius` 包的一部分。相反，您需要单独安装 `@mercuriusjs/gateway` 包：

```bash
$ npm install @mercuriusjs/gateway
```

同样，对于创建联合模式，您需要安装 `@mercuriusjs/federation` 包：

```bash
$ npm install @mercuriusjs/federation
```

### 从 v9 迁移到 v10

本章节提供了从 `@nestjs/graphql` 版本 9 迁移到版本 10 的一系列指南。这个主要版本发布的重点是提供一个更轻量级、平台无关的核心库。

#### 引入“驱动”包

在最新版本中，我们决定将 `@nestjs/graphql` 包拆分成几个独立的库，让您选择在项目中使用 Apollo（`@nestjs/apollo`）、Mercurius（`@nestjs/mercurius`）或其他 GraphQL 库。

这意味着现在您必须明确指定应用程序将使用哪种驱动。

```typescript
// 之前
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot({
      autoSchemaFile: 'schema.gql',
    }),
  ],
})
export class AppModule {}

// 之后
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: 'schema.gql',
    }),
  ],
})
export class AppModule {}
```

#### 插件

Apollo Server 插件允许您在响应某些事件时执行自定义操作。由于这是 Apollo 的独有特性，我们将其从 `@nestjs/graphql` 移动到了新创建的 `@nestjs/apollo` 包中，因此您需要更新应用程序中的导入。

```typescript
// 之前
import { Plugin } from '@nestjs/graphql';

// 之后
import { Plugin } from '@nestjs/apollo';
```

#### 指令

`schemaDirectives` 特性已被新的 [Schema directives API](https://www.graphql-tools.com/docs/schema-directives) 在 `@graphql-tools/schema` 包的 v8 中取代。

```typescript
// 之前
import { SchemaDirectiveVisitor } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLField } from 'graphql';

export class UpperCaseDirective extends SchemaDirectiveVisitor {
  visitFieldDefinition(field: GraphQLField<any, any>) {
    const { resolve = defaultFieldResolver } = field;
    field.resolve = async function (...args) {
      const result = await resolve.apply(this, args);
      if (typeof result === 'string') {
        return result.toUpperCase();
      }
      return result;
    };
  }
}

// 之后
import { getDirective, MapperKind, mapSchema } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLSchema } from 'graphql';

export function upperDirectiveTransformer(
  schema: GraphQLSchema,
  directiveName: string,
) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const upperDirective = getDirective(
        schema,
        fieldConfig,
        directiveName,
      )?.[0];

      if (upperDirective) {
        const { resolve = defaultFieldResolver } = fieldConfig;

        // 替换原始解析器，首先调用原始解析器，然后将结果转换为大写
        fieldConfig.resolve = async function (source, args, context, info) {
          const result = await resolve(source, args, context, info);
          if (typeof result === 'string') {
            return result.toUpperCase();
          }
          return result;
        };
        return fieldConfig;
      }
    },
  });
}

```

要将这个指令实现应用于包含 `@upper` 指令的模式，请使用 `transformSchema` 函数：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  ...
  transformSchema: schema => upperDirectiveTransformer(schema, 'upper'),
})
```

#### 联合

`GraphQLFederationModule` 已被移除，并被相应的驱动类取代：

```typescript
// 之前
GraphQLFederationModule.forRoot({
  autoSchemaFile: true,
});

// 之后
GraphQLModule.forRoot<ApolloFederationDriverConfig>({
  driver: ApolloFederationDriver,
  autoSchemaFile: true,
});
```

> info **提示** `ApolloFederationDriver` 类和 `ApolloFederationDriverConfig` 都从 `@nestjs/apollo` 包中导出。

同样，而不是使用专门的 `GraphQLGatewayModule`，只需将适当的 `driver` 类传递给您的 `GraphQLModule` 设置：

```typescript
// 之前
GraphQLGatewayModule.forRoot({
  gateway: {
    supergraphSdl: new IntrospectAndCompose({
      subgraphs: [
        { name: 'users', url: 'http://localhost:3000/graphql' },
        { name: 'posts', url: 'http://localhost:3001/graphql' },
      ],
    }),
  },
});

// 之后
GraphQLModule.forRoot<ApolloGatewayDriverConfig>({
  driver: ApolloGatewayDriver,
  gateway: {
    supergraphSdl: new IntrospectAndCompose({
      subgraphs: [
        { name: 'users', url: 'http://localhost:3000/graphql' },
        { name: 'posts', url: 'http://localhost:3001/graphql' },
      ],
    }),
  },
});
```

> info **提示** `ApolloGatewayDriver` 类和 `ApolloGatewayDriverConfig` 都从 `@nestjs/apollo` 包中导出。