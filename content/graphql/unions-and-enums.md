### 联合类型

联合类型与接口非常相似，但它们不能指定类型之间的任何共同字段（了解更多[这里](https://graphql.org/learn/schema/#union-types)）。联合类型适用于从单个字段返回不同的数据类型。

#### 代码优先

要定义GraphQL联合类型，我们必须定义这个联合将由哪些类组成。根据Apollo文档中的[示例](https://www.apollographql.com/docs/apollo-server/schema/unions-interfaces/#union-type)，我们将创建两个类。首先是`Book`：

```typescript
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Book {
  @Field()
  title: string;
}
```

然后是`Author`：

```typescript
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Author {
  @Field()
  name: string;
}
```

有了这些，使用`@nestjs/graphql`包导出的`createUnionType`函数注册`ResultUnion`联合：

```typescript
export const ResultUnion = createUnionType({
  name: 'ResultUnion',
  types: () => [Author, Book] as const,
});
```

> 警告 **警告** `createUnionType`函数的`types`属性返回的数组应该给出一个常量断言。如果没有给出常量断言，在编译时会生成错误的声明文件，并且在从另一个项目中使用时会发生错误。

现在，我们可以在查询中引用`ResultUnion`：

```typescript
@Query(() => [ResultUnion])
search(): Array<typeof ResultUnion> {
  return [new Author(), new Book()];
}
```

这将生成以下GraphQL模式的一部分SDL：

```graphql
type Author {
  name: String!
}

type Book {
  title: String!
}

union ResultUnion = Author | Book

type Query {
  search: [ResultUnion!]!
}
```

默认的`resolveType()`函数由库生成，将根据解析器方法返回的值提取类型。这意味着返回类实例而不是字面量JavaScript对象是强制性的。

要提供自定义的`resolveType()`函数，将`resolveType`属性传递给传递给`createUnionType()`函数的选项对象，如下所示：

```typescript
export const ResultUnion = createUnionType({
  name: 'ResultUnion',
  types: () => [Author, Book] as const,
  resolveType(value) {
    if (value.name) {
      return Author;
    }
    if (value.title) {
      return Book;
    }
    return null;
  },
});
```

#### 模式优先

要在模式优先方法中定义联合，只需用SDL创建GraphQL联合。

```graphql
type Author {
  name: String!
}

type Book {
  title: String!
}

union ResultUnion = Author | Book
```

然后，您可以使用类型生成功能（如[快速开始](/graphql/quick-start)章节所示）生成相应的TypeScript定义：

```typescript
export class Author {
  name: string;
}

export class Book {
  title: string;
}

export type ResultUnion = Author | Book;
```

联合需要在解析器映射中额外的`__resolveType`字段来确定联合应该解析到哪种类型。另外，请注意`ResultUnionResolver`类必须注册为任何模块中的提供者。让我们创建一个`ResultUnionResolver`类并定义`__resolveType`方法。

```typescript
@Resolver('ResultUnion')
export class ResultUnionResolver {
  @ResolveField()
  __resolveType(value) {
    if (value.name) {
      return 'Author';
    }
    if (value.title) {
      return 'Book';
    }
    return null;
  }
}
```

> 信息 **提示** 所有装饰器都从`@nestjs/graphql`包导出。

### 枚举

枚举类型是一种特殊的标量，它被限制在一组特定的允许值中（了解更多[这里](https://graphql.org/learn/schema/#enumeration-types)）。这允许您：

- 验证此类型的任何参数是否是允许的值之一
- 通过类型系统传达字段将始终是有限值集之一

#### 代码优先

当使用代码优先方法时，您可以通过简单地创建TypeScript枚举来定义GraphQL枚举类型。

```typescript
export enum AllowedColor {
  RED,
  GREEN,
  BLUE,
}
```

有了这些，使用`@nestjs/graphql`包导出的`registerEnumType`函数注册`AllowedColor`枚举：

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
});
```

现在您可以在我们的类型中引用`AllowedColor`：

```typescript
@Field(type => AllowedColor)
favoriteColor: AllowedColor;
```

这将生成以下GraphQL模式的一部分SDL：

```graphql
enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

要为枚举提供描述，将`description`属性传递给`registerEnumType()`函数。

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
  description: 'The supported colors.',
});
```

要为枚举值提供描述，或将一个值标记为弃用，传递`valuesMap`属性，如下所示：

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
  description: 'The supported colors.',
  valuesMap: {
    RED: {
      description: 'The default color.',
    },
    BLUE: {
      deprecationReason: 'Too blue.',
    },
  },
});
```

这将生成以下GraphQL模式的SDL：

```graphql
"""
The supported colors.
"""
enum AllowedColor {
  """
  The default color.
  """
  RED
  GREEN
  BLUE @deprecated(reason: "Too blue.")
}
```

#### 模式优先

要在模式优先方法中定义枚举器，只需用SDL创建GraphQL枚举。

```graphql
enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

然后您可以使用类型生成功能（如[快速开始](/graphql/quick-start)章节所示）生成相应的TypeScript定义：

```typescript
export enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

有时后端会强制使用与公共API不同的枚举值。在这个例子中，API包含`RED`，但在解析器中我们可能使用`#f00`代替（了解更多[这里](https://www.apollographql.com/docs/apollo-server/schema/scalars-enums/#internal-values)）。要实现这一点，为`AllowedColor`枚举声明一个解析器对象：

```typescript
export const allowedColorResolver: Record<keyof typeof AllowedColor, any> = {
  RED: '#f00',
};
```

> 信息 **提示** 所有装饰器都从`@nestjs/graphql`包导出。

然后使用这个解析器对象和`GraphQLModule#forRoot()`方法的`resolvers`属性一起使用，如下所示：

```typescript
GraphQLModule.forRoot({
  resolvers: {
    AllowedColor: allowedColorResolver,
  },
});
```