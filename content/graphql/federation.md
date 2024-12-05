### 联合

联合提供了一种将您的单体GraphQL服务器分割成独立的微服务的方法。它由两个组件组成：一个网关和一个或多个联合微服务。每个微服务持有部分架构，网关将架构合并成一个可以被客户端消费的单一架构。

引用自[Apollo文档](https://blog.apollographql.com/apollo-federation-f260cf525d21)，联合设计遵循以下核心原则：

- 构建图应该是**声明式的**。通过联合，你可以在架构内部声明式地组合一个图，而不是编写命令式的架构缝合代码。
- 代码应该按**关注点**分离，而不是按类型。通常没有单一团队控制像用户或产品这样的重要类型的每个方面，因此这些类型的的定义应该分布在不同的团队和代码库中，而不是集中管理。
- 图应该简单到客户端可以轻松消费。联合服务可以共同形成一个完整的、以产品为中心的图，准确反映客户端的消费方式。
- 它只是**GraphQL**，只使用语言的符合规范的特性。任何语言，不仅仅是JavaScript，都可以实现联合。

> 警告 **警告** 联合目前不支持订阅。

在接下来的部分中，我们将设置一个演示应用程序，它由一个网关和两个联合端点组成：用户服务和帖子服务。

#### 使用Apollo进行联合

首先，安装所需的依赖项：

```bash
$ npm install --save @apollo/subgraph
```

#### 架构优先

“用户服务”提供了一个简单的架构。注意`@key`指令：它指示Apollo查询规划器，如果指定了`id`，则可以获取`User`的特定实例。同时，注意我们`extend`了`Query`类型。

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

解析器提供了一个名为`resolveReference()`的额外方法。当Apollo网关需要一个用户实例时，该方法会被触发。稍后我们将在帖子服务中看到此方法的示例。请注意，该方法必须用`@ResolveReference()`装饰器标注。

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过在配置对象中传递`ApolloFederationDriver`驱动程序来注册`GraphQLModule`：

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### 代码优先

首先，向`User`实体添加一些额外的装饰器。

```ts
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field()
  name: string;
}
```

解析器提供了一个名为`resolveReference()`的额外方法。当Apollo网关需要一个用户实例时，该方法会被触发。稍后我们将在帖子服务中看到此方法的示例。请注意，该方法必须用`@ResolveReference()`装饰器标注。

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver(() => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query(() => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过在配置对象中传递`ApolloFederationDriver`驱动程序来注册`GraphQLModule`：

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

联合示例代码优先模式可在[这里](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/users-application)找到，架构优先模式可在[这里](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/users-application)找到。

#### 联合示例：帖子

帖子服务应该通过`getPosts`查询提供聚合的帖子，但同时也要通过`user.posts`字段扩展我们的`User`类型。

#### 架构优先

“帖子服务”通过在其架构中标记`extend`关键字来引用`User`类型。它还在`User`类型上声明了一个额外的属性（`posts`）。注意用于匹配User实例的`@key`指令，以及表示`id`字段在其他地方管理的`@external`指令。

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

以下示例中，`PostsResolver`提供了`getUser()`方法，该方法返回包含`__typename`和一些应用程序可能需要的其他属性的引用，以解析引用，在这种情况下是`id`。`__typename`由GraphQL网关使用，以确定负责User类型的微服务并检索相应的实例。上面描述的“用户服务”将在执行`resolveReference()`方法时被请求。

```typescript
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

最后，我们必须像在“用户服务”部分所做的那样注册`GraphQLModule`。

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolvers],
})
export class AppModule {}
```

#### 代码优先

首先，我们将声明一个代表`User`实体的类。尽管实体本身存在于另一个服务中，但我们将在这里使用它（扩展其定义）。注意`@extends`和`@external`指令。

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  @Directive('@external')
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

现在让我们为`User`实体的扩展创建相应的解析器，如下所示：

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField(() => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

我们还需要定义`Post`实体类：

```ts
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field(() => ID)
  id: number;

  @Field()
  title: string;

  @Field(() => Int)
  authorId: number;

  @Field(() => User)
  user?: User;
}
```

以及它的解析器：

```ts
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query(() => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query(() => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField(() => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

最后，在模块中将其整合在一起。注意架构构建选项，我们指定`User`是一个孤立（外部）类型。

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

代码优先模式的联合示例可在[这里](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/posts-application)找到，架构优先模式的联合示例可在[这里](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/posts-application)找到。

#### 联合示例：网关

首先，安装所需的依赖项：

```bash
$ npm install --save @apollo/gateway
```

网关需要指定一个端点列表，它将自动发现相应的架构。因此，无论是代码优先还是架构优先方法，网关服务的实现都将保持不变。

```typescript
import { IntrospectAndCompose } from '@apollo/gateway';
import { ApolloGatewayDriver, ApolloGatewayDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloGatewayDriverConfig>({
      driver: ApolloGatewayDriver,
      server: {
        // ... Apollo 服务器选项
        cors: true,
      },
      gateway: {
        supergraphSdl: new IntrospectAndCompose({
          subgraphs: [
            { name: 'users', url: 'http://user-service/graphql' },
            { name: 'posts', url: 'http://post-service/graphql' },
          ],
        }),
      },
    }),
  ],
})
export class AppModule {}
```

代码优先模式的联合示例可在[这里](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/gateway)找到，架构优先模式的联合示例可在[这里](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/gateway)找到。

#### 与Mercurius联合

首先，安装所需的依赖项：

```bash
$ npm install --save @apollo/subgraph @nestjs/mercurius
```

> 信息 **注意** `@apollo/subgraph`包是构建子图架构（`buildSubgraphSchema`, `printSubgraphSchema`函数）所必需的。

#### 架构优先

“用户服务”提供了一个简单的架构。注意`@key`指令：它指示Mercurius查询规划器，如果指定了`id`，则可以获取`User`的特定实例。同时，注意我们`extend`了`Query`类型。

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

解析器提供了一个名为`resolveReference()`的额外方法。当Mercurius网关需要一个用户实例时，该方法会被触发。稍后我们将在帖子服务中看到此方法的示例。请注意，该方法必须用`@ResolveReference()`装饰器标注。

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过在配置对象中传递`MercuriusFederationDriver`驱动程序来注册`GraphQLModule`：

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      typePaths: ['**/*.graphql'],
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### 代码优先

首先，向`User`实体添加一些额外的装饰器。

```ts
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field()
  name: string;
}
```

解析器提供了一个名为`resolveReference()`的额外方法。当Mercurius网关需要一个用户实例时，该方法会被触发。稍后我们将在帖子服务中看到此方法的示例。请注意，该方法必须用`@ResolveReference()`装饰器标注。

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver(() => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query(() => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过在配置对象中传递`MercuriusFederationDriver`驱动程序来注册`GraphQLModule`：

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### 联合示例：帖子

帖子服务应该通过`getPosts`查询提供聚合的帖子，但同时也要通过`user.posts`字段扩展我们的`User`类型。

#### 架构优先

“帖子服务”通过在其架构中标记`extend`关键字来引用`User`类型。它还在`User`类型上声明了一个额外的属性（`posts`）。注意用于匹配User实例的`@key`指令，以及表示`id`字段在其他地方管理的`@external`指令。

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

以下示例中，`PostsResolver`提供了`getUser()`方法，该方法返回包含`__typename`和一些应用程序可能需要的其他属性的引用，以解析引用，在这种情况下是`id`。`__typename`由GraphQL网关使用，以确定负责User类型的微服务并检索相应的实例。上面描述的“用户服务”将在执行`resolveReference()`方法时被请求。

```typescript
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

最后，我们必须像在“用户服务”部分所做的那样注册`GraphQLModule`。

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      federationMetadata: true,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolvers],
})
export class AppModule {}
```

#### 代码优先

首先，我们将声明一个代表`User`实体的类。尽管实体本身存在于另一个服务中，但我们将在这里使用它（扩展其定义）。注意`@extends`和`@external`指令。

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  @Directive('@external')
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

现在让我们为`User`实体的扩展创建相应的解析器，如下所示：

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField(() => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

我们还需要定义`Post`实体类：

```ts
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field(() => ID)
  id: number;

  @Field()
  title: string;

  @Field(() => Int)
  authorId: number;

  @Field(() => User)
  user?: User;
}
```

以及它的解析器：

```ts
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query(() => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query(() => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField(() => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

最后，在模块中将其整合在一起。注意架构构建选项，我们指定`User`是一个孤立（外部）类型。

```ts
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

#### 联合2

引用自[Apollo文档](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2)，联合2在原始Apollo联合（本文档中称为联合1）的基础上改进了开发者体验，与大多数原始超图向后兼容。

> 警告 **警告** Mercurius不完全支持联合2。你可以在[这里](https://www.apollographql.com/docs/federation/supported-subgraphs#javascript--typescript)查看支持联合2的库列表。

在接下来的部分中，我们将升级前面的示例到联合2。

#### 联合示例：用户

联合2的一个变化是实体没有原始子图，所以我们不再需要扩展`Query`了。更详细的信息请参考[实体主题](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2#entities)在Apollo联合2文档中。

#### 架构优先

我们可以简单地从架构中移除`extend`关键字。

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

type Query {
  getUser(id: ID!): User
}
```

#### 代码优先

要使用联合2，我们需要在`autoSchemaFile`选项中指定联合版本。

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### 联合示例：帖子

由于上述原因，我们不再需要扩展`User`和`Query`了。

#### 架构优先

我们可以简单地从架构中移除`extend`和`external`指令。

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

type User @key(fields: "id") {
  id: ID!
  posts: [Post]
}

type Query {
  getPosts: [Post]
}
```

#### 代码优先

由于我们不再扩展`User`实体了，我们可以简单地从`User`中移除`extends`和`external`指令。

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

同样，像用户服务一样，我们需要在`GraphQLModule`中指定使用联合2。

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```