### 变异（Mutations）

大多数关于GraphQL的讨论都集中在数据获取上，但任何完整的数据平台都需要一种修改服务器端数据的方法。在REST中，任何请求都可能导致服务器上的副作用，但最佳实践建议我们不应该在GET请求中修改数据。GraphQL类似——技术上任何查询都可以实现导致数据写入。然而，像REST一样，建议遵循约定，即任何导致写入的操作都应通过变异（mutation）显式发送（了解更多[这里](https://graphql.org/learn/queries/#mutations)）。

官方的[Apollo](https://www.apollographql.com/docs/graphql-tools/generate-schema.html)文档使用了一个`upvotePost()`变异示例。这个变异实现了一个方法来增加帖子的`votes`属性值。要在Nest中创建一个等效的变异，我们将使用`@Mutation()`装饰器。

#### 代码优先

让我们在上一节中使用的`AuthorResolver`中添加另一个方法（参见[解析器](/graphql/resolvers)）。

```typescript
@Mutation(() => Post)
async upvotePost(@Args({ name: 'postId', type: () => Int }) postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

> 信息提示：所有装饰器（例如`@Resolver`、`@ResolveField`、`@Args`等）都是从`@nestjs/graphql`包中导出的。

这将在SDL中生成以下GraphQL模式部分：

```graphql
type Mutation {
  upvotePost(postId: Int!): Post
}
```

`upvotePost()`方法接受`postId`（`Int`）作为参数，并返回一个更新后的`Post`实体。由于在[解析器](/graphql/resolvers)部分解释的原因，我们需要显式设置预期的类型。

如果变异需要接受一个对象作为参数，我们可以创建一个**输入类型**。输入类型是一种特殊类型的对象类型，可以作为参数传递（了解更多[这里](https://graphql.org/learn/schema/#input-types)）。要声明一个输入类型，使用`@InputType()`装饰器。

```typescript
import { InputType, Field } from '@nestjs/graphql';

@InputType()
export class UpvotePostInput {
  @Field()
  postId: number;
}
```

> 信息提示：`@InputType()`装饰器接受一个选项对象作为参数，因此你可以，例如，指定输入类型的描述。请注意，由于TypeScript的元数据反射系统限制，你必须使用`@Field`装饰器手动指示类型，或使用[CLI插件](/graphql/cli-plugin)。

然后我们可以使用这个类型在解析器类中：

```typescript
@Mutation(() => Post)
async upvotePost(
  @Args('upvotePostData') upvotePostData: UpvotePostInput,
) {}
```

#### 模式优先

让我们扩展上一节中使用的`AuthorResolver`（参见[解析器](/graphql/resolvers)）。

```typescript
@Mutation()
async upvotePost(@Args('postId') postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

请注意，我们假设上述业务逻辑已经移动到`PostsService`（查询帖子并增加其`votes`属性）。`PostsService`类中的逻辑可以简单也可以复杂，根据需要。这个示例的主要目的是展示解析器如何与其他提供者交互。

最后一步是将我们的变异添加到现有的类型定义中。

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}

type Mutation {
  upvotePost(postId: Int!): Post
}
```

`upvotePost(postId: Int!): Post`变异现在可以作为我们应用程序的GraphQL API的一部分被调用。