### 指令

指令可以附加到字段或片段包含上，并且可以以服务器希望的任何方式影响查询的执行（了解更多[这里](https://graphql.org/learn/queries/#directives)）。GraphQL规范提供了几个默认指令：

- `@include(if: Boolean)` - 如果参数为真，则仅在结果中包含此字段
- `@skip(if: Boolean)` - 如果参数为真，则跳过此字段
- `@deprecated(reason: String)` - 用消息标记字段为已弃用

指令是一个以`@`字符开头的标识符，后面可以跟一个命名参数列表，它可以出现在GraphQL查询和模式语言的几乎所有元素之后。

#### 自定义指令

要指示当Apollo/Mercurius遇到你的指令时应该发生什么，你可以创建一个转换函数。这个函数使用`mapSchema`函数遍历你的模式中的位置（字段定义、类型定义等）并执行相应的转换。

```typescript
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

        // 替换原始解析器为一个函数，该函数首先调用原始解析器，然后将结果转换为大写
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

现在，在`GraphQLModule#forRoot`方法中使用`transformSchema`函数应用`upperDirectiveTransformer`转换函数：

```typescript
GraphQLModule.forRoot({
  // ...
  transformSchema: (schema) => upperDirectiveTransformer(schema, 'upper'),
});
```

注册后，`@upper`指令可以在我们的模式中使用。但是，你应用指令的方式将取决于你使用的方法（代码优先或模式优先）。

#### 代码优先

在代码优先方法中，使用`@Directive()`装饰器应用指令。

```typescript
@Directive('@upper')
@Field()
title: string;
```

> 提示 **提示** `@Directive()`装饰器是从`@nestjs/graphql`包中导出的。

指令可以应用于字段、字段解析器、输入和对象类型，以及查询、突变和订阅。这里是一个在查询处理器级别应用指令的例子：

```typescript
@Directive('@deprecated(reason: "This query will be removed in the next version")')
@Query(() => Author, { name: 'author' })
async getAuthor(@Args({ name: 'id', type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

> 警告 **警告** 通过`@Directive()`装饰器应用的指令将不会反映在生成的模式定义文件中。

最后，确保在`GraphQLModule`中声明指令，如下所示：

```typescript
GraphQLModule.forRoot({
  // ...,
  transformSchema: schema => upperDirectiveTransformer(schema, 'upper'),
  buildSchemaOptions: {
    directives: [
      new GraphQLDirective({
        name: 'upper',
        locations: [DirectiveLocation.FIELD_DEFINITION],
      }),
    ],
  },
}),
```

> 提示 **提示** 两者`GraphQLDirective`和`DirectiveLocation`都是从`graphql`包中导出的。

#### 模式优先

在模式优先方法中，直接在SDL中应用指令。

```graphql
directive @upper on FIELD_DEFINITION

type Post {
  id: Int!
  title: String! @upper
  votes: Int
}
```