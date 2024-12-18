Nest 应用程序处理请求并产生响应的过程被称为**请求生命周期**。通过使用中间件、管道、守卫和拦截器，尤其是在全局、控制器级别和路由级别组件发挥作用时，追踪特定代码在请求生命周期中执行的位置可能会变得具有挑战性。通常情况下，请求会流经中间件到守卫，然后到拦截器，接着到管道，最后在返回路径上（即在生成响应时）再回到拦截器。

#### 中间件

中间件按照特定的顺序执行。首先，Nest 运行全局绑定的中间件（例如使用 `app.use` 绑定的中间件），然后运行[模块绑定的中间件](/middleware)，这些中间件是根据路径确定的。中间件按照绑定的顺序依次运行，类似于 Express 中间件的工作方式。对于绑定在不同模块的中间件，绑定到根模块的中间件将首先运行，然后按照模块被添加到导入数组的顺序运行中间件。

#### 守卫

守卫的执行从全局守卫开始，然后是控制器守卫，最后是路由守卫。与中间件一样，守卫按照它们绑定的顺序运行。例如：

```typescript
@UseGuards(Guard1, Guard2)
@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @UseGuards(Guard3)
  @Get()
  getCats(): Cats[] {
    return this.catsService.getCats();
  }
}
```

`Guard1` 将在 `Guard2` 之前执行，两者都将在 `Guard3` 之前执行。

> 信息 **提示** 当谈论全局绑定与控制器或局部绑定的区别时，差异在于守卫（或其他组件）绑定的位置。如果你使用 `app.useGlobalGuard()` 或通过模块提供组件，它是全局绑定的。否则，如果装饰器在控制器类之前，它绑定到控制器；如果装饰器在路由声明之前，它绑定到路由。

#### 拦截器

拦截器在很大程度上遵循与守卫相同的模式，但有一个例外：由于拦截器返回的是[RxJS Observables](https://github.com/ReactiveX/rxjs)，这些可观察对象将以先进后出的顺序被解析。因此，入站请求将通过标准的全局、控制器、路由级别解析，但请求的响应端（即从控制器方法处理器返回后）将从路由到控制器再到全局进行解析。此外，任何由管道、控制器或服务抛出的错误都可以在拦截器的 `catchError` 操作符中读取。

#### 管道

管道遵循标准的全局到控制器到路由绑定的顺序，关于 `@UsePipes()` 参数，同样遵循先进先出的顺序。然而，在路由参数级别，如果有多个管道运行，它们将按照最后一个参数带有管道到第一个参数的顺序运行。这也适用于路由级别和控制器级别的管道。例如，如果我们有以下控制器：

```typescript
@UsePipes(GeneralValidationPipe)
@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @UsePipes(RouteSpecificPipe)
  @Patch(':id')
  updateCat(
    @Body() body: UpdateCatDTO,
    @Param() params: UpdateCatParams,
    @Query() query: UpdateCatQuery,
  ) {
    return this.catsService.updateCat(body, params, query);
  }
}
```

那么 `GeneralValidationPipe` 将先于 `query`，然后是 `params`，接着是 `body` 对象运行，然后再运行 `RouteSpecificPipe`，顺序相同。如果有任何参数特定的管道，它们将在控制器和路由级别的管道之后运行（同样，从最后一个参数到第一个参数）。

#### 过滤器

过滤器是唯一不首先解析全局的组件。相反，过滤器从尽可能低的级别解析，这意味着执行从任何路由绑定的过滤器开始，接下来是控制器级别，最后是全局过滤器。注意，异常不能从一个过滤器传递到另一个过滤器；如果路由级别的过滤器捕获了异常，控制器或全局级别的过滤器就不能捕获同一个异常。实现类似效果的唯一方法是在过滤器之间使用继承。

> 信息 **提示** 过滤器只有在请求处理过程中发生任何未捕获的异常时才执行。使用 `try/catch` 捕获的异常，例如，不会触发异常过滤器。一旦遇到未捕获的异常，其余的生命周期将被忽略，请求直接跳转到过滤器。

#### 总结

总的来说，请求生命周期如下：

1. 入站请求
2. 中间件
   - 2.1 全局绑定中间件
   - 2.2 模块绑定中间件
3. 守卫
   - 3.1 全局守卫
   - 3.2 控制器守卫
   - 3.3 路由守卫
4. 拦截器（控制器前）
   - 4.1 全局拦截器
   - 4.2 控制器拦截器
   - 4.3 路由拦截器
5. 管道
   - 5.1 全局管道
   - 5.2 控制器管道
   - 5.3 路由管道
   - 5.4 路由参数管道
6. 控制器（方法处理器）
7. 服务（如果存在）
8. 拦截器（请求后）
   - 8.1 路由拦截器
   - 8.2 控制器拦截器
   - 8.3 全局拦截器
9. 异常过滤器
   - 9.1 路由
   - 9.2 控制器
   - 9.3 全局
10. 服务器响应