在GraphQL世界中，关于如何处理诸如**认证**或操作的**副作用**等问题有很多争论。我们应该在业务逻辑内部处理这些事情吗？我们应该使用高阶函数来增强查询和变更操作的授权逻辑吗？还是应该使用[模式指令](https://www.apollographql.com/docs/apollo-server/schema/directives/)？这些问题没有单一的万能答案。

Nest通过其跨平台特性如[守卫](/guards)和[拦截器](/interceptors)来帮助解决这些问题。理念是减少冗余并提供工具，帮助创建结构良好、可读性强和一致性高的应用。

#### 概览

你可以像处理任何RESTful应用一样，在GraphQL中使用标准的[守卫](/guards)、[拦截器](/interceptors)、[过滤器](/exception-filters)和[管道](/pipes)。此外，你还可以利用[自定义装饰器](/custom-decorators)功能轻松创建自己的装饰器。让我们来看一个GraphQL查询处理器的示例。

```typescript
@Query('author')
@UseGuards(AuthGuard)
async getAuthor(@Args('id', ParseIntPipe) id: number) {
  return this.authorsService.findOneById(id);
}
```

如你所见，GraphQL与守卫和管道的协作方式与HTTP REST处理器相同。因此，你可以将认证逻辑移动到守卫中；你甚至可以在REST和GraphQL API接口之间重用相同的守卫类。同样，拦截器在两种类型的应用中以相同的方式工作：

```typescript
@Mutation()
@UseInterceptors(EventsInterceptor)
async upvotePost(@Args('postId') postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

#### 执行上下文

由于GraphQL接收的请求数据类型与REST不同，因此[执行上下文](https://docs.nestjs.com/fundamentals/execution-context)在GraphQL与REST之间有所不同。GraphQL解析器有一组独特的参数：`root`、`args`、`context`和`info`。因此，守卫和拦截器必须将通用的`ExecutionContext`转换为`GqlExecutionContext`。这很简单：

```typescript
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const ctx = GqlExecutionContext.create(context);
    return true;
  }
}
```

`GqlExecutionContext.create()`返回的GraphQL上下文对象暴露了每个GraphQL解析器参数的**get**方法（例如`getArgs()`、`getContext()`等）。一旦转换，我们可以轻松地为当前请求挑选出任何GraphQL参数。

#### 异常过滤器

Nest标准的[异常过滤器](/exception-filters)也与GraphQL应用兼容。与`ExecutionContext`一样，GraphQL应用应该将`ArgumentsHost`对象转换为`GqlArgumentsHost`对象。

```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements GqlExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const gqlHost = GqlArgumentsHost.create(host);
    return exception;
  }
}
```

> 信息**提示** `GqlExceptionFilter`和`GqlArgumentsHost`都从`@nestjs/graphql`包中导入。

请注意，与REST情况不同，你不使用原生的`response`对象来生成响应。

#### 自定义装饰器

如上所述，[自定义装饰器](/custom-decorators)功能在GraphQL解析器中按预期工作。

```typescript
export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) =>
    GqlExecutionContext.create(ctx).getContext().user,
);
```

如下使用`@User()`自定义装饰器：

```typescript
@Mutation()
async upvotePost(
  @User() user: UserEntity,
  @Args('postId') postId: number,
) {}
```

> 信息**提示** 在上述示例中，我们假设`user`对象被分配给了你的GraphQL应用的上下文。

#### 在字段解析器级别执行增强器

在GraphQL上下文中，Nest不会在字段级别运行**增强器**（拦截器、守卫和过滤器的通用名称）[参见此问题](https://github.com/nestjs/graphql/issues/320#issuecomment-511193229)：它们仅对顶级`@Query()`/`@Mutation()`方法运行。你可以通过在`GqlModuleOptions`中设置`fieldResolverEnhancers`选项来告诉Nest为用`@ResolveField()`注解的方法执行拦截器、守卫或过滤器。适当地传递一个`'interceptors'`、`'guards'`和/或`'filters'`列表：

```typescript
GraphQLModule.forRoot({
  fieldResolverEnhancers: ['interceptors']
}),
```

> **警告** 为字段解析器启用增强器可能会导致性能问题，当你返回大量记录并且你的字段解析器被执行数千次时。因此，当你启用`fieldResolverEnhancers`时，我们建议你跳过对字段解析器不必要的增强器的执行。你可以使用以下辅助函数来实现这一点：

```typescript
export function isResolvingGraphQLField(context: ExecutionContext): boolean {
  if (context.getType<GqlContextType>() === 'graphql') {
    const gqlContext = GqlExecutionContext.create(context);
    const info = gqlContext.getInfo();
    const parentType = info.parentType.name;
    return parentType !== 'Query' && parentType !== 'Mutation';
  }
  return false;
}
```

#### 创建自定义驱动程序

Nest提供了两个官方驱动程序：`@nestjs/apollo`和`@nestjs/mercurius`，以及一个API，允许开发人员构建新的**自定义驱动程序**。有了自定义驱动程序，你可以集成任何GraphQL库或扩展现有集成，增加额外的功能。

例如，要集成`express-graphql`包，你可以创建以下驱动程序类：

```typescript
import { AbstractGraphQLDriver, GqlModuleOptions } from '@nestjs/graphql';
import { graphqlHTTP } from 'express-graphql';

class ExpressGraphQLDriver extends AbstractGraphQLDriver {
  async start(options: GqlModuleOptions<any>): Promise<void> {
    options = await this.graphQlFactory.mergeWithSchema(options);

    const { httpAdapter } = this.httpAdapterHost;
    httpAdapter.use(
      '/graphql',
      graphqlHTTP({
        schema: options.schema,
        graphiql: true,
      }),
    );
  }

  async stop() {}
}

```

然后可以这样使用：

```typescript
GraphQLModule.forRoot({
  driver: ExpressGraphQLDriver,
});
```