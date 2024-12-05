### 接口

像许多类型系统一样，GraphQL支持接口。**接口**是一种抽象类型，它包括了实现接口的类型必须包含的一组字段（更多信息[点击这里](https://graphql.org/learn/schema/#interfaces)）。

#### 代码优先

在使用代码优先方法时，您可以通过创建一个带有`@InterfaceType()`装饰器的抽象类来定义GraphQL接口，该装饰器是从`@nestjs/graphql`导出的。

```typescript
import { Field, ID, InterfaceType } from '@nestjs/graphql';

@InterfaceType()
export abstract class Character {
  @Field(() => ID)
  id: string;

  @Field()
  name: string;
}
```

> 警告 **警告** TypeScript接口不能用来定义GraphQL接口。

这将在SDL中生成以下GraphQL模式部分：

```graphql
interface Character {
  id: ID!
  name: String!
}
```

现在，要实现`Character`接口，请使用`implements`关键字：

```typescript
@ObjectType({
  implements: () => [Character],
})
export class Human implements Character {
  id: string;
  name: string;
}
```

> 提示 **提示** `@ObjectType()`装饰器是从`@nestjs/graphql`包导出的。

默认的`resolveType()`函数由库生成，它基于解析器方法返回的值提取类型。这意味着您必须返回类实例（您不能返回字面JavaScript对象）。

要提供自定义的`resolveType()`函数，请将`resolveType`属性传递给传入`@InterfaceType()`装饰器的选项对象，如下所示：

```typescript
@InterfaceType({
  resolveType(book) {
    if (book.colors) {
      return ColoringBook;
    }
    return TextBook;
  },
})
export abstract class Book {
  @Field(() => ID)
  id: string;

  @Field()
  title: string;
}
```

#### 接口解析器

到目前为止，使用接口，您只能与对象共享字段定义。如果您还想要共享实际的字段解析器实现，可以创建一个专用的接口解析器，如下所示：

```typescript
import { Resolver, ResolveField, Parent, Info } from '@nestjs/graphql';

@Resolver((type) => Character) // 提醒：Character是一个接口
export class CharacterInterfaceResolver {
  @ResolveField(() => [Character])
  friends(
    @Parent() character, // 解析的对象，实现了Character
    @Info() { parentType }, // 实现了Character的对象类型
    @Args('search', { type: () => String }) searchTerm: string,
  ) {
    // 获取角色的朋友
    return [];
  }
}
```

现在`friends`字段解析器自动注册到所有实现`Character`接口的对象类型。

> 警告 **警告** 这需要在`GraphQLModule`配置中将`inheritResolversFromInterfaces`属性设置为true。

#### 模式优先

要在模式优先方法中定义接口，只需使用SDL创建GraphQL接口。

```graphql
interface Character {
  id: ID!
  name: String!
}
```

然后，您可以使用类型生成功能（如[快速开始](/graphql/quick-start)章节所示）来生成相应的TypeScript定义：

```typescript
export interface Character {
  id: string;
  name: string;
}
```

接口需要在解析器映射中额外添加一个`__resolveType`字段，以确定接口应该解析到哪种类型。让我们创建一个`CharactersResolver`类并定义`__resolveType`方法：

```typescript
@Resolver('Character')
export class CharactersResolver {
  @ResolveField()
  __resolveType(value) {
    if ('age' in value) {
      return Person;
    }
    return null;
  }
}
```

> 提示 **提示** 所有装饰器都是从`@nestjs/graphql`包导出的。