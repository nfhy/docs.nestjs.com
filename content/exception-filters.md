### 异常过滤器

Nest 提供了一个内置的**异常层**，负责处理应用程序中的所有未处理异常。当异常没有被应用程序代码处理时，它会被这一层捕获，然后自动发送适当的用户友好响应。

<figure>
  <img class="illustrative-image" src="/assets/Filter_1.png" />
</figure>

默认情况下，这个操作由内置的**全局异常过滤器**执行，它处理类型为 `HttpException`（及其子类的）异常。当异常**无法识别**（既不是 `HttpException` 也不是继承自 `HttpException` 的类）时，内置异常过滤器会生成以下默认 JSON 响应：

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> **提示** 全局异常过滤器部分支持 `http-errors` 库。基本上，任何抛出的异常包含 `statusCode` 和 `message` 属性都将被正确填充并作为响应发送回去（而不是对于无法识别的异常默认的 `InternalServerErrorException`）。

#### 抛出标准异常

Nest 提供了一个内置的 `HttpException` 类，从 `@nestjs/common` 包中导出。对于典型的 HTTP REST/GraphQL API 应用程序，在某些错误条件发生时发送标准 HTTP 响应对象是最佳实践。

例如，在 `CatsController` 中，我们有一个 `findAll()` 方法（一个 `GET` 路由处理器）。假设这个路由处理器由于某种原因抛出了异常。为了演示这一点，我们将如下硬编码：

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

> **提示** 这里使用了 `HttpStatus`。这是一个从 `@nestjs/common` 包导入的帮助枚举。

当客户端调用这个端点时，响应如下所示：

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

`HttpException` 构造函数接受两个必需的参数，这些参数决定了响应：

- `response` 参数定义 JSON 响应体。它可以是一个 `string` 或如下所述的 `object`。
- `status` 参数定义 [HTTP 状态码](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)。

默认情况下，JSON 响应体包含两个属性：

- `statusCode`：默认为 `status` 参数中提供的 HTTP 状态码。
- `message`：基于 `status` 的 HTTP 错误的简短描述。

要覆盖 JSON 响应体的仅消息部分，提供字符串作为 `response` 参数。要覆盖整个 JSON 响应体，传递一个对象作为 `response` 参数。Nest 将序列化对象并将其作为 JSON 响应体返回。

第二个构造函数参数 - `status` - 应该是一个有效的 HTTP 状态码。最佳实践是使用从 `@nestjs/common` 导入的 `HttpStatus` 枚举。

还有一个 **第三** 个构造函数参数（可选）- `options` - 可用于提供错误 [cause](https://nodejs.org/en/blog/release/v16.9.0/#error-cause)。这个 `cause` 对象不会序列化到响应对象中，但对于日志记录目的很有用，提供了导致抛出 `HttpException` 的内部错误的有价值信息。

以下是覆盖整个响应体并提供错误原因的示例：

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) {
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```

使用上述代码，响应将如下所示：

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```

#### 自定义异常

在许多情况下，您不需要编写自定义异常，可以使用下一部分描述的内置 Nest HTTP 异常。如果您确实需要创建自定义异常，最佳实践是创建自己的 **异常层次结构**，让您的自定义异常继承自基础 `HttpException` 类。通过这种方法，Nest 将识别您的异常，并自动处理错误响应。让我们实现这样一个自定义异常：

```typescript
@@filename(forbidden.exception)
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

由于 `ForbiddenException` 扩展了基础 `HttpException`，它将与内置异常处理器无缝工作，因此我们可以使用它在 `findAll()` 方法中。

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

#### 内置 HTTP 异常

Nest 提供了一组从基础 `HttpException` 继承的标准异常。这些异常从 `@nestjs/common` 包中导出，代表了大多数常见的 HTTP 异常：

- `BadRequestException`
- `UnauthorizedException`
- `NotFoundException`
- `ForbiddenException`
- `NotAcceptableException`
- `RequestTimeoutException`
- `ConflictException`
- `GoneException`
- `HttpVersionNotSupportedException`
- `PayloadTooLargeException`
- `UnsupportedMediaTypeException`
- `UnprocessableEntityException`
- `InternalServerErrorException`
- `NotImplementedException`
- `ImATeapotException`
- `MethodNotAllowedException`
- `BadGatewayException`
- `ServiceUnavailableException`
- `GatewayTimeoutException`
- `PreconditionFailedException`

所有内置异常也可以使用 `options` 参数提供错误 `cause` 和错误描述：

```typescript
throw new BadRequestException('Something bad happened', {
  cause: new Error(),
  description: 'Some error description',
});
```

使用上述代码，响应将如下所示：

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400
}
```

#### 异常过滤器

虽然基础（内置）异常过滤器可以自动处理许多情况，但您可能想要对异常层有 **完全控制**。例如，您可能想要添加日志记录或根据一些动态因素使用不同的 JSON 架构。**异常过滤器** 正是为此目的而设计的。它们让您控制响应发送回客户端的确切流程和内容。

让我们创建一个异常过滤器，负责捕获 `HttpException` 类的实例，并为它们实现自定义响应逻辑。为此，我们需要访问底层平台的 `Request` 和 `Response` 对象。我们访问 `Request` 对象是为了提取原始的 `url` 并将其包含在日志信息中。我们将使用 `Response` 对象直接控制发送的响应，使用 `response.json()` 方法。

```typescript
@@filename(http-exception.filter)
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
@@switch
import { Catch, HttpException } from '@nestjs/common';

@Catch(HttpException)
export class HttpExceptionFilter {
  catch(exception, host) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

> **提示** 所有异常过滤器都应该实现泛型 `ExceptionFilter<T>` 接口。这要求您提供带有指定签名的 `catch(exception: T, host: ArgumentsHost)` 方法。`T` 表示异常的类型。

> **警告** 如果您使用的是 `@nestjs/platform-fastify`，您可以使用 `response.send()` 代替 `response.json()`。不要忘记从 `fastify` 导入正确的类型。

`@Catch(HttpException)` 装饰器将所需的元数据绑定到异常过滤器，告诉 Nest 这个特定的过滤器正在寻找 `HttpException` 类型的异常，没有其他。`@Catch()` 装饰器可以带一个参数，或者一个逗号分隔的列表。这让您可以一次为几种类型的异常设置过滤器。

#### 参数主机

让我们看看 `catch()` 方法的参数。`exception` 参数是当前正在处理的异常对象。`host` 参数是一个 `ArgumentsHost` 对象。`ArgumentsHost` 是一个强大的实用程序对象，我们将在 [执行上下文章节](/fundamentals/execution-context) 中进一步检查。在此代码示例中，我们使用它来获取传递给原始请求处理程序的 `Request` 和 `Response` 对象的引用（在控制器中异常起源的地方）。在此代码示例中，我们使用了 `ArgumentsHost` 上的一些帮助方法来获取所需的 `Request` 和 `Response` 对象。在 [这里](/fundamentals/execution-context) 了解更多关于 `ArgumentsHost` 的信息。

*“ArgumentsHost”之所以有这种抽象级别，是因为它在所有上下文中都起作用（例如，我们现在工作的 HTTP 服务器上下文，但也包括微服务和 WebSocket）。在执行上下文章节中，我们将看到如何使用 `ArgumentsHost` 及其帮助函数访问 **任何** 执行上下文的适当 <a href="https://docs.nestjs.com/fundamentals/execution-context#host-methods">底层参数</a>。这将使我们能够编写跨所有上下文操作的通用异常过滤器。*

#### 绑定过滤器

让我们将新的 `HttpExceptionFilter` 绑定到 `CatsController` 的 `create()` 方法。

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(new HttpExceptionFilter())
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> **提示** `@UseFilters()` 装饰器从 `@nestjs/common` 包导入。

这里我们使用了 `@UseFilters()` 装饰器。类似于 `@Catch()` 装饰器，它可以带一个过滤器实例，或者一个逗号分隔的过滤器实例列表。在这里，我们当场创建了 `HttpExceptionFilter` 的实例。或者，您可以传递类（而不是实例），将实例化的责任留给框架，并启用 **依赖注入**。

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(HttpExceptionFilter)
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> **提示** 尽可能优先使用类而不是实例来应用过滤器。它减少了 **内存使用**，因为 Nest 可以轻松地在您的整个模块中重用同一个类的实例。

在上面的例子中，`HttpExceptionFilter` 只应用于单个 `create()` 路由处理器，使其成为方法范围的。异常过滤器可以有不同的范围：方法范围的控制器/解析器/网关，控制器范围的，或全局范围的。

例如，要将过滤器设置为控制器范围的，您将如下操作：

```typescript
@@filename(cats.controller)
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

这种构造为 `CatsController` 中定义的每个路由处理器设置了 `HttpExceptionFilter`。

要创建一个全局范围的过滤器，您将如下操作：

```typescript
@@filename(main)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> **警告** `useGlobalFilters()` 方法不设置网关或混合应用程序的过滤器。

全局范围的过滤器在整个应用程序中使用，对于每个控制器和每个路由处理器。就依赖注入而言，从模块外部注册的全局过滤器（如上例中的 `useGlobalFilters()`）不能注入依赖项，因为这是在任何模块的上下文之外完成的。为了解决这个问题，您可以直接从任何模块使用以下构造注册全局范围的过滤器：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

> **提示** 使用这种方法进行依赖注入时，请注意，无论在哪个模块使用这种构造，过滤器实际上是全局的。应该在哪里完成？选择定义过滤器的模块（上例中的 `HttpExceptionFilter`）。此外，`useClass` 不是处理自定义提供者注册的唯一方式。了解更多 [here](/fundamentals/custom-providers)。

您可以使用这种技术添加尽可能多的过滤器；只需将每个过滤器添加到提供者数组中。

#### 捕获一切

为了捕获 **所有** 未处理的异常（无论异常类型如何），在 `@Catch()` 装饰器的参数列表中留空，例如 `@Catch()`。

在下面的示例中，我们有一段代码是平台无关的，因为它使用 [HTTP 适配器](./faq/http-adapter) 来传递响应，并且没有直接使用任何平台特定的对象（`Request` 和 `Response`）：

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class CatchEverythingFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // 在某些情况下 `httpAdapter` 可能在构造函数方法中不可用，因此我们应该在这里解析它。
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> **警告** 当将捕获一切的异常过滤器与绑定到特定类型的过滤器结合使用时，“捕获任何”过滤器应该先声明，以允许特定过滤器正确处理绑定的类型。

#### 继承

通常，您将创建完全定制的异常过滤器，以满足您的应用程序需求。然而，可能有一些用例，您希望简单地扩展内置默认的 **全局异常过滤器**，并根据某些因素覆盖行为。

为了将异常处理委托给基础过滤器，您需要扩展 `BaseExceptionFilter` 并调用继承的 `catch()` 方法。

```typescript
@@filename(all-exceptions.filter)
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

> **警告** 方法范围和控制器范围的过滤器扩展了 `BaseExceptionFilter` 不应该用 `new` 实例化。相反，让框架自动实例化它们。

全局过滤器 **可以** 扩展基础过滤器。这可以通过两种方式完成。

第一种方法是在实例化自定义全局过滤器时注入 `HttpAdapter` 引用：

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

第二种方法是使用 `APP_FILTER` 标记 <a href="exception-filters#binding-filters">如这里所示</a>。