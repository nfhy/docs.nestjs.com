### 解析器

解析器提供了将 GraphQL 操作（查询、变更或订阅）转换为数据的指令。它们返回我们在模式中指定的相同形状的数据——要么同步返回，要么作为承诺返回该形状的结果。通常，您会手动创建一个**解析器映射**。另一方面，`@nestjs/graphql` 包使用您用于注释类的装饰器提供的元数据自动生成解析器映射。为了演示如何使用包功能创建 GraphQL API 的过程，我们将创建一个简单的作者 API。

#### 代码优先

在代码优先方法中，我们不遵循通过手写 GraphQL SDL 创建 GraphQL 模式的典型过程。相反，我们使用 TypeScript 装饰器从 TypeScript 类定义生成 SDL。`@nestjs/graphql` 包读取通过装饰器定义的元数据，并为您自动生成模式。

#### 对象类型

GraphQL 模式中的大多数定义都是**对象类型**。您定义的每个对象类型应该代表应用程序客户端可能需要与之交互的域对象。例如，我们的示例 API 需要能够获取作者列表及其帖子，因此我们应该定义 `Author` 类型和 `Post` 类型以支持此功能。

如果我们使用模式优先方法，我们会像这样用 SDL 定义这样的模式：

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post!]!
}
```

在这种情况下，使用代码优先方法，我们使用 TypeScript 类和 TypeScript 装饰器来注释这些类的字段。上述 SDL 在代码优先方法中的等效是：

```typescript
@@filename(authors/models/author.model)
import { Field, Int, ObjectType } from '@nestjs/graphql';
import { Post } from './post';

@ObjectType()
export class Author {
  @Field(type => Int)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

> 信息 **提示** TypeScript 的元数据反射系统有几个限制，这使得它无法确定类包含哪些属性，或者是否识别给定属性是可选的还是必需的。由于这些限制，我们必须在模式定义类中显式使用 `@Field()` 装饰器来提供有关每个字段的 GraphQL 类型和可选性的元数据，或者使用 [CLI 插件](/graphql/cli-plugin) 为我们生成这些。

`Author` 对象类型，像任何类一样，由一系列字段组成，每个字段声明一个类型。字段的类型对应于 [GraphQL 类型](https://graphql.org/learn/schema/)。字段的 GraphQL 类型可以是另一个对象类型或标量类型。GraphQL 标量类型是一个原始类型（如 `ID`、`String`、`Boolean` 或 `Int`），它解析为单个值。

> 信息 **提示** 除了 GraphQL 的内置标量类型外，您还可以定义自定义标量类型（阅读[更多](/graphql/scalars)）。

上述 `Author` 对象类型定义将导致 Nest 生成我们上面显示的 SDL：

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post!]!
}
```

`@Field()` 装饰器接受一个可选的类型函数（例如，`type => Int`），以及可选的选项对象。

当存在 TypeScript 类型系统和 GraphQL 类型系统之间的歧义时，类型函数是必需的。具体来说：对于 `string` 和 `boolean` 类型**不**需要它；对于 `number`（必须映射到 GraphQL `Int` 或 `Float`）**需要**它。类型函数应该简单地返回所需的 GraphQL 类型（如这些章节中的各种示例所示）。

选项对象可以有以下任一键/值对：

- `nullable`：指定字段是否可为空（在 `@nestjs/graphql` 中，每个字段默认不可为空）；`boolean`
- `description`：设置字段描述；`string`
- `deprecationReason`：将字段标记为已弃用；`string`

例如：

```typescript
@Field({ description: `书名`, deprecationReason: '在 v2 模式中无用' })
title: string;
```

> 信息 **提示** 您还可以为整个对象类型添加描述，或弃用它：`@ObjectType({{ '{' }} description: '作者模型' {{ '}' }})`。

当字段是数组时，我们必须在 `Field()` 装饰器的类型函数中手动指示数组类型，如下所示：

```typescript
@Field(type => [Post])
posts: Post[];
```

> 信息 **提示** 使用数组括号符号（`[ ]`），我们可以指示数组的深度。例如，使用 `[[Int]]` 将表示一个整数矩阵。

要声明数组项（而不是数组本身）可以为空，请如下所示将 `nullable` 属性设置为 `'items'`：

```typescript
@Field(type => [Post], { nullable: 'items' })
posts: Post[];
```

> 信息 **提示** 如果数组及其项都可以为空，请将 `nullable` 设置为 `'itemsAndList'`。

现在 `Author` 对象类型已经创建，我们定义 `Post` 对象类型。

```typescript
@@filename(posts/models/post.model)
import { Field, Int, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Post {
  @Field(type => Int)
  id: number;

  @Field()
  title: string;

  @Field(type => Int, { nullable: true })
  votes?: number;
}
```

`Post` 对象类型将在 SDL 中生成以下 GraphQL 模式部分：

```graphql
type Post {
  id: Int!
  title: String!
  votes: Int
}
```

#### 代码优先解析器

到目前为止，我们已经定义了可以存在于我们数据图中的对象（类型定义），但客户端尚未有与这些对象交互的方法。为了解决这个问题，我们需要创建一个解析器类。在代码优先方法中，解析器类既定义解析器函数**又**生成**查询类型**。这将在下面的示例中变得清晰：

```typescript
@@filename(authors/authors.resolver)
@Resolver(() => Author)
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query(() => Author)
  async author(@Args('id', { type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField()
  async posts(@Parent() author: Author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> 信息 **提示** 所有装饰器（例如，`@Resolver`，`@ResolveField`，`@Args` 等）都从 `@nestjs/graphql` 包中导出。

您可以定义多个解析器类。Nest 将在运行时将这些组合在一起。有关代码组织方面的更多信息，请参阅下面的[模块](/graphql/resolvers#module)部分。

> 警告 **注意** `AuthorsService` 和 `PostsService` 类中的逻辑可以简单或复杂，根据需要。此示例的主要目的是展示如何构建解析器以及它们如何与其他提供者交互。

在上面的示例中，我们创建了 `AuthorsResolver`，它定义了一个查询解析器函数和一个字段解析器函数。要创建解析器，我们创建一个类，将解析器函数作为方法，并使用 `@Resolver()` 装饰器注释该类。

在这个示例中，我们定义了一个查询处理器来根据请求中发送的 `id` 获取作者对象。要指定该方法是查询处理器，请使用 `@Query()` 装饰器。

传递给 `@Resolver()` 装饰器的参数是可选的，但在图变得非平凡时就发挥作用。它用于提供解析器函数使用的父对象，因为它们遍历对象图。在我们的示例中，由于类包括一个**字段解析器**函数（用于 `Author` 对象类型的 `posts` 属性），我们**必须**向 `@Resolver()` 装饰器提供一个值，以指示此类中定义的所有字段解析器的父类型（即相应的 `ObjectType` 类名）。从示例中应该清楚的是，当编写字段解析器函数时，需要访问父对象（字段被解析为其成员的对象）。在这个示例中，我们使用字段解析器填充作者的帖子数组，该字段解析器调用一个服务，该服务以作者的 `id` 作为参数。因此需要在 `@Resolver()` 装饰器中识别父对象。请注意相应的 `@Parent()` 方法参数装饰器的使用，以便在字段解析器中提取对父对象的引用。

我们可以定义多个 `@Query()` 解析器函数（在此类中以及任何其他解析器类中），它们将被聚合到生成的 SDL 中的单个**查询类型**定义中，以及解析器映射中的适当条目中。这允许您将查询定义靠近它们使用的模型和服务，并在模块中保持良好组织。

> 信息 **提示** Nest CLI 提供了一个生成器（schematic），它自动生成**所有样板代码**，以帮助我们避免做所有这些工作，并使开发人员体验更简单。在[这里](/recipes/crud-generator)阅读有关此功能的更多信息。

#### 查询类型名称

在上面的示例中，`@Query()` 装饰器基于方法名称生成 GraphQL 模式查询类型名称。例如，考虑上面的示例中的以下构造：

```typescript
@Query(() => Author)
async author(@Args('id', { type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

这将在我们模式中生成以下作者查询条目（查询类型使用与方法名称相同的名称）：

```graphql
type Query {
  author(id: Int!): Author
}
```

> 信息 **提示** 了解更多关于 GraphQL 查询的信息[在这里](https://graphql.org/learn/queries/)。

按照惯例，我们更倾向于将这些名称分开；例如，我们更倾向于使用像 `getAuthor()` 这样的名称作为我们的查询处理器方法，但仍然使用 `author` 作为我们的查询类型名称。我们的字段解析器也是如此。我们可以通过将映射名称作为 `@Query()` 和 `@ResolveField()` 装饰器的参数来轻松实现这一点，如下所示：

```typescript
@@filename(authors/authors.resolver)
@Resolver(() => Author)
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query(() => Author, { name: 'author' })
  async getAuthor(@Args('id', { type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField('posts', () => [Post])
  async getPosts(@Parent() author: Author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

上述 `getAuthor` 处理器方法将在 SDL 中生成以下部分的 GraphQL 模式：

```graphql
type Query {
  author(id: Int!): Author
}
```

#### 查询装饰器选项

`@Query()` 装饰器的选项对象（我们在上面传递 `{{ '{' }}name: 'author'{{ '}' }}`）接受许多键/值对：

- `name`：查询的名称；一个 `string`
- `description`：将用于生成 GraphQL 模式文档的描述（例如，在 GraphQL playground 中）；一个 `string`
- `deprecationReason`：将查询元数据设置为弃用（例如，在 GraphQL playground 中）；一个 `string`
- `nullable`：查询是否可以返回空数据响应；`boolean` 或 `'items'` 或 `'itemsAndList'`（有关 `'items'` 和 `'itemsAndList'` 的详细信息，请参见上文）

#### 参数装饰器选项

使用 `@Args()` 装饰器从请求中提取参数以供方法处理器使用。这与 [REST 路由参数参数提取](/controllers#route-parameters)的工作方式非常相似。

通常，您的 `@Args()` 装饰器将很简单，不需要像 `getAuthor()` 方法中那样的对象参数。例如，如果标识符的类型是字符串，以下构造就足够了，并且简单地从传入的 GraphQL 请求中提取命名字段以用作方法参数。

```typescript
@Args('id') id: string
```

在 `getAuthor()` 案例中，使用了 `number` 类型，这带来了挑战。`number` TypeScript 类型没有给我们足够的信息来表示预期的 GraphQL 表示（例如，`Int` 对比 `Float`）。因此，我们必须**显式**传递类型引用。我们通过向 `Args()` 装饰器传递第二个参数来包含参数选项，如下所示：

```typescript
@Query(() => Author, { name: 'author' })
async getAuthor(@Args('id', { type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

选项对象允许我们指定以下可选的键值对：

- `type`：返回 GraphQL 类型的函数
- `defaultValue`：默认值；`any`
- `description`：描述元数据；`string`
- `deprecationReason`：弃用字段并提供描述原因的元数据；`string`
- `nullable`：字段是否可为空

查询处理器方法可以接收多个参数。假设我们想根据其 `firstName` 和 `lastName` 获取作者。在这种情况下，我们可以调用 `@Args` 两次：

```typescript
getAuthor(
  @Args('firstName', { nullable: true }) firstName?: string,
  @Args('lastName', { defaultValue: '' }) lastName?: string,
) {}
```

#### 专用参数类

使用内联 `@Args()` 调用，像上面的示例中的代码变得膨胀。相反，您可以创建一个专用的 `GetAuthorArgs` 参数类，并在处理器方法中按如下方式访问它：

```typescript
@Args() args: GetAuthorArgs
```

使用 `@ArgsType()` 创建 `GetAuthorArgs` 类，如下所示：

```typescript
@@filename(authors/dto/get-author.args)
import { MinLength } from 'class-validator';
import { Field, ArgsType } from '@nestjs/graphql';

@ArgsType()
class GetAuthorArgs {
  @Field({ nullable: true })
  firstName?: string;

  @Field({ defaultValue: '' })
  @MinLength(3)
  lastName: string;
}
```

> 信息 **提示** 再次，由于 TypeScript 的元数据反射系统的限制，需要使用 `@Field` 装饰器手动指示类型和可选性，或使用 [CLI 插件](/graphql/cli-plugin)。

这将在 SDL 中生成以下部分的 GraphQL 模式：

```graphql
type Query {
  author(firstName: String, lastName: String = ''): Author
}
```

> 信息 **提示** 请注意，参数类如 `GetAuthorArgs` 与 `ValidationPipe` 配合得很好（阅读[更多](/techniques/validation)）。

#### 类继承

您可以使用标准 TypeScript 类继承来创建具有通用实用程序类型功能的基类（字段和字段属性、验证等），这些功能可以被扩展。例如，您可能有一组始终包括标准 `offset` 和 `limit` 字段的分页相关参数，但也包括其他特定于类型的索引字段。您可以如下所示设置类层次结构。

基 `@ArgsType()` 类：

```typescript
@ArgsType()
class PaginationArgs {
  @Field(() => Int)
  offset: number = 0;

  @Field(() => Int)
  limit: number = 10;
}
```

基 `@ArgsType()` 类的特定于类型的子类：

```typescript
@ArgsType()
class GetAuthorArgs extends PaginationArgs {
  @Field({ nullable: true })
  firstName?: string;

  @Field({ defaultValue: '' })
  @MinLength(3)
  lastName: string;
}
```

与 `@ObjectType()` 对象一样，可以使用继承。在基类上定义通用属性：

```typescript
@ObjectType()
class Character {
  @Field(() => Int)
  id: number;

  @Field()
  name: string;
}
```

在子类上添加特定于类型的属性：

```typescript
@ObjectType()
class Warrior extends Character {
  @Field()
  level: number;
}
```

您也可以使用继承与解析器。您可以通过结合继承和 TypeScript 泛型来确保类型安全。例如，要创建一个具有通用 `findAll` 查询的基类，使用如下构造：

```typescript
function BaseResolver<T extends Type<unknown>>(classRef: T): any {
  @Resolver({ isAbstract: true })
  abstract class BaseResolverHost {
    @Query(() => [classRef], { name: `findAll${classRef.name}` })
    async findAll(): Promise<T[]> {
      return [];
    }
  }
  return BaseResolverHost;
}
```

请注意以下几点：

- 需要显式返回类型（上述的 `any`）：否则 TypeScript 会抱怨使用私有类定义的使用。建议：定义一个接口而不是使用 `any`。
- `Type` 从 `@nestjs/common` 包导入
- `isAbstract: true` 属性表明不应为此类生成 SDL（模式定义语言语句）。注意，您也可以为其他类型设置此属性以抑制 SDL 生成。

以下是如何生成 `BaseResolver` 的具体子类：

```typescript
@Resolver(() => Recipe)
export class RecipesResolver extends BaseResolver(Recipe) {
  constructor(private recipesService: RecipesService) {
    super();
  }
}
```

这种构造将生成以下 SDL：

```graphql
type Query {
  findAllRecipe: [Recipe!]!
}
```

#### 泛型

我们在上述内容中看到了泛型的一种用途。这个强大的 TypeScript 功能可以用来创建有用的抽象。例如，以下是基于 [此文档](https://graphql.org/learn/pagination/#pagination-and-edges) 的示例，实现了基于游标的分页实现：

```typescript
import { Field, ObjectType, Int } from '@nestjs/graphql';
import { Type } from '@nestjs/common';

interface IEdgeType<T> {
  cursor: string;
  node: T;
}

export interface IPaginatedType<T> {
  edges: IEdgeType<T>[];
  nodes: T[];
  totalCount: number;
  hasNextPage: boolean;
}

export function Paginated<T>(classRef: Type<T>): Type<IPaginatedType<T>> {
  @ObjectType(`${classRef.name}Edge`)
  abstract class EdgeType {
    @Field(() => String)
    cursor: string;

    @Field(() => classRef)
    node: T;
  }

  @ObjectType({ isAbstract: true })
  abstract class PaginatedType implements IPaginatedType<T> {
    @Field(() => [EdgeType], { nullable: true })
    edges: EdgeType[];

    @Field(() => [classRef], { nullable: true })
    nodes: T[];

    @Field(() => Int)
    totalCount: number;

    @Field()
    hasNextPage: boolean;
  }
  return PaginatedType as Type<IPaginatedType<T>>;
}
```

使用上述基类定义，我们现在可以轻松地创建继承此行为的专用类型。例如：

```typescript
@ObjectType()
class PaginatedAuthor extends Paginated(Author) {}
```

#### 模式优先

如在[前一](/graphql/quick-start)章节中提到的，在模式优先方法中，我们首先手动在 SDL 中定义模式类型（阅读[更多](https://graphql.org/learn/schema/#type-language)）。考虑以下 SDL 类型定义。

> 信息 **提示** 为了方便本章，我们已经将所有 SDL 汇总在一个位置（例如，一个 `.graphql` 文件，如下所示）。在实践中，您可能会发现以模块化方式组织代码是适当的。例如，通过在专用目录中为每个域实体创建单独的 SDL 文件，其中包含类型定义、相关服务、解析器代码和 Nest 模块定义类，可能会有所帮助。Nest 将在运行时聚合所有单独的模式类型定义。

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String!
  votes: Int
}

type Query {
  author(id: Int!): Author
}
```

#### 模式优先解析器

上述模式暴露了一个查询 - `author(id: Int!): Author`。

> 信息 **提示** 在[这里](https://graphql.org/learn/queries/)了解更多关于 GraphQL 查询的信息。

现在让我们创建一个 `AuthorsResolver` 类来解析作者查询：

```typescript
@@filename(authors/authors.resolver)
@Resolver('Author')
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query()
  async author(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField()
  async posts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> 信息 **提示** 所有装饰器（例如，`@Resolver`，`@ResolveField`，`@Args` 等）都从 `@nestjs/graphql` 包中导出。

> 警告 **注意** `AuthorsService` 和 `PostsService` 类中的逻辑可以简单或复杂，根据需要。此示例的主要目的是展示如何构建解析器以及它们如何与其他提供者交互。

`@Resolver()` 装饰器是必需的。它接受一个可选的字符串参数，即类名。这个类名是必需的，只要类中包含 `@ResolveField()` 装饰器，以通知 Nest 该装饰的方法与父类型（我们当前示例中的 `Author` 类型）相关联。或者，而不是在类顶部设置 `@Resolver()`，这可以为每个方法完成：

```typescript
@Resolver('Author')
@ResolveField()
async posts(@Parent() author) {
  const { id } = author;
  return this.postsService.findAll({ authorId: id });
}
```

在这种情况下（在方法级别使用 `@Resolver()` 装饰器），如果您的类中有多个 `@ResolveField()` 装饰器，您必须将 `@Resolver()` 添加到所有这些装饰器中。这不是最佳实践（因为它创建了额外的开销）。

> 信息 **提示** 传递给 `@Resolver()` 的任何类名参数**不**影响查询（`@Query()` 装饰器）或变更（`@Mutation()` 装饰器）。

> 警告 **警告** 使用方法级别的 `@Resolver` 装饰器在 **代码优先** 方法中不受支持。

在上面的示例中，`@Query()` 和 `@ResolveField()` 装饰器基于方法名称与 GraphQL 模式类型相关联。例如，考虑上面的示例中的以下构造：

```typescript
@Query()
async author(@Args('id') id: number) {
  return this.authorsService.findOneById(id);
}
```

这将在我们模式中生成以下作者查询条目（查询类型使用与方法名称相同的名称）：

```graphql
type Query {
  author(id: Int!): Author
}
```

按照惯例，我们更倾向于将这些分开，使用像 `getAuthor()` 或 `getPosts()` 这样的名称作为我们的解析器方法。我们可以通过将映射名称作为参数传递给装饰器，如下所示：

```typescript
@@filename(authors/authors.resolver)
@Resolver('Author')
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query('author')
  async getAuthor(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField('posts')
  async getPosts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> 信息 **提示** Nest CLI 提供了一个生成器（schematic），它自动生成**所有样板代码**，以帮助我们避免做所有这些工作，并使开发人员体验更简单。在[这里](/recipes/crud-generator)阅读有关此功能的更多信息。

#### 生成类型

假设我们使用模式优先方法，并启用了类型生成功能（`outputAs: 'class'` 如在[前一](/graphql/quick-start)章节中所示），一旦您运行应用程序，它将生成以下文件（在 `GraphQLModule.forRoot()` 方法中指定的位置）。例如，在 `src/graphql.ts` 中：

```typescript
@@filename(graphql)
export (class Author {
  id: number;
  firstName?: string;
  lastName?: string;
  posts?: Post[];
})

export class Post {
  id: number;
  title: string;
  votes?: number;
}

export abstract class IQuery {
  abstract author(id: number): Author | Promise<Author>;
}
```

通过生成类（而不是默认技术生成接口），您可以使用声明性验证**装饰器**与模式优先方法结合使用，这是一种极其有用的技术（阅读[更多](/techniques/validation)）。例如，您可以向生成的 `CreatePostInput` 类添加 `class-validator` 装饰器，如下所示，以强制执行 `title` 字段的最小和最大字符串长度：

```typescript
import { MinLength, MaxLength } from 'class-validator';

export class CreatePostInput {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

> 警告 **注意** 要启用输入（和参数）的自动验证，请使用 `ValidationPipe`。在[这里](/techniques/validation)了解更多关于验证的信息，以及更具体地了解管道[在这里](/pipes)。

但是，如果您直接在自动生成的文件上添加装饰器，它们将每次生成文件时被**覆盖**。相反，创建一个单独的文件，只需扩展生成的类。

```typescript
import { MinLength, MaxLength } from 'class-validator';
import { Post } from '../../graphql.ts';

export class CreatePostInput extends Post {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

#### GraphQL 参数装饰器

我们可以使用专用装饰器访问标准 GraphQL 解析器参数。以下是 Nest 装饰器和平纹 Apollo 参数的比较。

<table>
  <tbody>
    <tr>
      <td><code>@Root()</code> 和 <code>@Parent()</code></td>
      <td><code>root</code>/ <code>parent</code></td>
    </tr>
    <tr>
      <td><code>@Context(param?: string)</code></td>
      <td><code>context</code> / <code>context[param]</code></td>
    </tr>
    <tr>
      <td><code>@Info(param?: string)</code></td>
      <td><code>info</code> / <code>info[param]</code></td>
    </tr>
    <tr>
      <td><code>@Args(param?: string)</code></td>
      <td><code>args</code> / <code>args[param]</code></td>
    </tr>
  </tbody>
</table>

这些参数具有以下含义：

- `root`：包含父字段解析器返回的结果的对象，或者在顶级 `Query` 字段的情况下，从服务器配置传递的 `rootValue`。
- `context`：所有解析器在特定查询中共享的对象；通常用于包含每个请求的状态。
- `info`：包含有关查询执行状态的信息的对象。
- `args`：包含查询中传递到字段的参数的对象。

#### 模块

完成上述步骤后，我们已经通过装饰器提供的元数据，以声明性方式指定了 `GraphQLModule` 生成解析器映射所需的所有信息。

您需要做的另一件事是**提供**（即，在某个模块中将解析器类（`AuthorsResolver`）列为 `provider`）并导入模块（`AuthorsModule`），以便 Nest 能够使用它。

例如，您可以在 `AuthorsModule` 中这样做，它也可以提供此上下文中需要的其他服务。确保在某个地方导入 `AuthorsModule`（例如，在根模块中，或由根模块导入的某个模块中）。

```typescript
@@filename(authors/authors.module)
@Module({
  imports: [PostsModule],
  providers: [AuthorsService, AuthorsResolver],
})
export class AuthorsModule {}
```

> 信息 **提示** 按所谓的**域模型**组织代码很有帮助（类似于您组织 REST API 的入口点的方式）。在这种方法中，将模型（`ObjectType` 类）、解析器和服务保持在一起，在一个代表域模型的 Nest 模块中。将所有这些组件放在每个模块的单独文件夹中。当您这样做时，并使用 [Nest CLI](/cli/overview) 生成每个元素，Nest 将自动为您连接所有这些部分（在适当的文件夹中定位文件，在 `provider` 和 `imports` 数组中生成条目等）。