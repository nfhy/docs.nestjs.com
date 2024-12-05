### 复杂性

> 警告 **警告** 本章节仅适用于代码优先方法。

查询复杂性允许您定义某些字段的复杂程度，并限制具有**最大复杂性**的查询。其思想是通过使用一个简单的数字来定义每个字段的复杂性。一个常见的默认值是给每个字段分配一个复杂性为`1`。此外，可以使用所谓的复杂性估算器自定义GraphQL查询的复杂性计算。复杂性估算器是一个简单的函数，用于计算字段的复杂性。您可以向规则中添加任意数量的复杂性估算器，然后依次执行它们。返回数值复杂性值的第一个估算器决定了该字段的复杂性。

`@nestjs/graphql`包与[graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity)等工具集成得非常好，该库提供了基于成本分析的解决方案。使用这个库，您可以拒绝执行被认为成本过高的GraphQL服务器查询。

#### 安装

要开始使用它，我们首先安装所需的依赖项。

```bash
$ npm install --save graphql-query-complexity
```

#### 开始使用

安装过程完成后，我们可以定义`ComplexityPlugin`类：

```typescript
import { GraphQLSchemaHost } from "@nestjs/graphql";
import { Plugin } from "@nestjs/apollo";
import {
  ApolloServerPlugin,
  GraphQLRequestListener,
} from 'apollo-server-plugin-base';
import { GraphQLError } from 'graphql';
import {
  fieldExtensionsEstimator,
  getComplexity,
  simpleEstimator,
} from 'graphql-query-complexity';

@Plugin()
export class ComplexityPlugin implements ApolloServerPlugin {
  constructor(private gqlSchemaHost: GraphQLSchemaHost) {}

  async requestDidStart(): Promise<GraphQLRequestListener> {
    const maxComplexity = 20;
    const { schema } = this.gqlSchemaHost;

    return {
      async didResolveOperation({ request, document }) {
        const complexity = getComplexity({
          schema,
          operationName: request.operationName,
          query: document,
          variables: request.variables,
          estimators: [
            fieldExtensionsEstimator(),
            simpleEstimator({ defaultComplexity: 1 }),
          ],
        });
        if (complexity > maxComplexity) {
          throw new GraphQLError(
            `查询过于复杂：${complexity}。允许的最大复杂性：${maxComplexity}`,
          );
        }
        console.log('查询复杂性：', complexity);
      },
    };
  }
}
```

为了演示目的，我们将允许的最大复杂性指定为`20`。在上面的例子中，我们使用了2个估算器，`simpleEstimator`和`fieldExtensionsEstimator`。

- `simpleEstimator`：简单估算器为每个字段返回固定的复杂性
- `fieldExtensionsEstimator`：字段扩展估算器提取您模式中每个字段的复杂性值

> 信息 **提示** 记得在任何模块的提供者数组中添加这个类。

#### 字段级复杂性

有了这个插件，我们现在可以通过在传入`@Field()`装饰器的选项对象中指定`complexity`属性来定义任何字段的复杂性，如下所示：

```typescript
@Field({ complexity: 3 })
title: string;
```

或者，您可以定义估算器函数：

```typescript
@Field({ complexity: (options: ComplexityEstimatorArgs) => ... })
title: string;
```

#### 查询/突变级复杂性

此外，`@Query()`和`@Mutation()`装饰器可以像这样指定`complexity`属性：

```typescript
@Query({ complexity: (options: ComplexityEstimatorArgs) => options.args.count * options.childComplexity })
items(@Args('count') count: number) {
  return this.itemsService.getItems({ count });
}
```