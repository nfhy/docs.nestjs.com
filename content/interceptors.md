### 拦截器

拦截器是一个用 `@Injectable()` 装饰器注解的类，并且实现了 `NestInterceptor` 接口。

<figure><img class="illustrative-image" src="/assets/Interceptors_1.png" /></figure>

拦截器具有一系列受 [面向切面编程](https://en.wikipedia.org/wiki/Aspect-oriented_programming) (AOP) 技术启发的有用功能。它们使得以下操作成为可能：

- 在方法执行前后绑定额外的逻辑
- 转换函数返回的结果
- 转换函数抛出的异常
- 扩展基本函数行为
- 根据特定条件（例如，出于缓存目的）完全覆盖函数

#### 基础

每个拦截器实现了 `intercept()` 方法，该方法接受两个参数。第一个是 `ExecutionContext` 实例（与 [守卫](/guards) 中的完全相同的对象）。`ExecutionContext` 继承自 `ArgumentsHost`。我们在异常过滤器章节之前看到过 `ArgumentsHost`。在那里，我们看到它是一个包装传递给原始处理器的参数的包装器，并包含基于应用程序类型的不同参数数组。你可以回顾[异常过滤器](https://docs.nestjs.com/exception-filters#arguments-host)以了解更多关于此主题的信息。

#### 执行上下文

通过扩展 `ArgumentsHost`，`ExecutionContext` 还添加了几个新的辅助方法，这些方法提供了有关当前执行过程的额外详细信息。这些详细信息可以帮助构建可以跨广泛的控制器、方法和执行上下文工作的更通用的拦截器。在[这里](/fundamentals/execution-context)了解更多关于 `ExecutionContext` 的信息。

#### 调用处理器

第二个参数是一个 `CallHandler`。`CallHandler` 接口实现了 `handle()` 方法，你可以在拦截器中的某个点使用它来调用路由处理器方法。如果你在 `intercept()` 方法的实现中没有调用 `handle()` 方法，路由处理器方法根本就不会被执行。

这种方法意味着 `intercept()` 方法有效地 **包装** 了请求/响应流。因此，你可以在最终路由处理器执行 **之前和之后** 实现自定义逻辑。很明显，你可以在调用 `handle()` 之前在 `intercept()` 方法中编写代码，但是你如何影响之后发生的事情呢？因为 `handle()` 方法返回一个 `Observable`，我们可以使用强大的 [RxJS](https://github.com/ReactiveX/rxjs) 操作符进一步操作响应。使用面向切面编程术语，路由处理器的调用（即，调用 `handle()`）被称为 [Pointcut](https://en.wikipedia.org/wiki/Pointcut)，表示这是我们插入额外逻辑的点。

考虑一个传入的 `POST /cats` 请求。这个请求是针对 `CatsController` 中定义的 `create()` 处理器的。如果在任何地方调用了不调用 `handle()` 方法的拦截器，`create()` 方法将不会被执行。一旦调用了 `handle()`（并且它的 `Observable` 已经被返回），`create()` 处理器将被触发。一旦通过 `Observable` 接收到响应流，可以在流上执行额外的操作，并将最终结果返回给调用者。

<app-banner-devtools></app-banner-devtools>

#### 切面拦截

我们将要查看的第一个用例是使用拦截器记录用户交互（例如，存储用户调用，异步分派事件或计算时间戳）。我们在下面展示了一个简单的 `LoggingInterceptor`：

```typescript
@@filename(logging.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor {
  intercept(context, next) {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

> 信息提示 **提示** `NestInterceptor<T, R>` 是一个泛型接口，其中 `T` 表示 `Observable<T>` 的类型（支持响应流），`R` 是 `Observable<R>` 包装的值的类型。

> 注意 **注意** 拦截器像控制器、提供者、守卫等一样，可以通过它们的 `constructor` **注入依赖**。

由于 `handle()` 返回一个 RxJS `Observable`，我们有广泛的操作符可供选择，用于操作流。在上面的例子中，我们使用了 `tap()` 操作符，它在 observable 流正常或异常终止时调用我们的匿名日志记录函数，但不会以其他方式干扰响应周期。

#### 绑定拦截器

为了设置拦截器，我们使用从 `@nestjs/common` 包导入的 `@UseInterceptors()` 装饰器。像 [管道](/pipes) 和 [守卫](/guards) 一样，拦截器可以是控制器范围的、方法范围的或全局范围的。

```typescript
@@filename(cats.controller)
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

> 信息提示 **提示** `@UseInterceptors()` 装饰器是从 `@nestjs/common` 包导入的。

使用上述构造，`CatsController` 中定义的每个路由处理器都将使用 `LoggingInterceptor`。当有人调用 `GET /cats` 端点时，你将在标准输出中看到以下输出：

```typescript
Before...
After... 1ms
```

请注意，我们传递了 `LoggingInterceptor` 类（而不是实例），将实例化的责任留给了框架，并启用了依赖注入。像管道、守卫和异常过滤器一样，我们也可以传递一个就地实例：

```typescript
@@filename(cats.controller)
@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

如上所述，上述构造将拦截器附加到此控制器声明的每个处理器上。如果我们想要将拦截器的作用域限制在单个方法上，我们只需在 **方法级别** 应用装饰器。

为了设置一个全局拦截器，我们使用 Nest 应用程序实例的 `useGlobalInterceptors()` 方法：

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

全局拦截器用于整个应用程序，对于每个控制器和每个路由处理器。在依赖注入方面，从模块外部注册的全局拦截器（如上例所示，使用 `useGlobalInterceptors()`）不能注入依赖项，因为这是在任何模块的上下文之外完成的。为了解决这个问题，你可以直接从任何模块设置拦截器，使用以下构造：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

> 信息提示 **提示** 当使用这种方法为拦截器执行依赖注入时，请注意，无论这种构造在哪个模块中使用，拦截器实际上都是全局的。应该在哪里做这件事？选择拦截器（如上例中的 `LoggingInterceptor`）定义的模块。此外，`useClass` 不是处理自定义提供者注册的唯一方式。在[这里](/fundamentals/custom-providers)了解更多。

#### 响应映射

我们已经知道 `handle()` 返回一个 `Observable`。流包含从路由处理器返回的值，因此我们可以使用 RxJS 的 `map()` 操作符轻松地对其进行变异。

> 警告 **警告** 响应映射功能不适用于库特定的响应策略（直接使用 `@Res()` 对象是禁止的）。

让我们创建一个 `TransformInterceptor`，它将以简单的方式修改每个响应以演示过程。它将使用 RxJS 的 `map()` 操作符将响应对象分配给新创建对象的 `data` 属性，将新对象返回给客户端。

```typescript
@@filename(transform.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor {
  intercept(context, next) {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

> 信息提示 **提示** Nest 拦截器适用于同步和异步 `intercept()` 方法。如果需要，你只需将方法切换为 `async`。

通过上述构造，当有人调用 `GET /cats` 端点时，响应将如下所示（假设路由处理器返回一个空数组 `[]`）：

```json
{
  "data": []
}
```

拦截器在创建整个应用程序中出现的通用需求的可重用解决方案方面具有很大的价值。例如，想象一下我们需要将每个 `null` 值转换为一个空字符串 `''`。我们可以用一行代码完成，并全局绑定拦截器，以便它将自动被每个注册的处理程序使用。

```typescript
@@filename()
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```

#### 异常映射

另一个有趣的用例是利用 RxJS 的 `catchError()` 操作符来覆盖抛出的异常：

```typescript
@@filename(errors.interceptor)
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
@@switch
import { Injectable, BadGatewayException } from '@nestjs/common';
import { throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```

#### 流覆盖

有时我们可能想要完全阻止调用处理器，并返回一个不同的值。一个明显的例子是实现缓存以提高响应时间。让我们看看一个简单的 **缓存拦截器**，它从缓存中返回其响应。在现实的例子中，我们想要考虑其他因素，如 TTL、缓存失效、缓存大小等，但这超出了这次讨论的范围。在这里，我们将提供一个基本的例子来演示主要概念。

```typescript
@@filename(cache.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { of } from 'rxjs';

@Injectable()
export class CacheInterceptor {
  intercept(context, next) {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

我们的 `CacheInterceptor` 有一个硬编码的 `isCached` 变量和一个硬编码的响应 `[]`。需要注意的关键是我们在这里返回了一个新的流，由 RxJS 的 `of()` 操作符创建，因此路由处理器 **根本不会被调用**。当有人调用使用 `CacheInterceptor` 的端点时，响应（一个硬编码的空数组）将立即返回。为了创建一个通用的解决方案，你可以利用 `Reflector` 创建一个自定义装饰器。`Reflector` 在 [守卫](/guards) 章节中有详细描述。

#### 更多操作符

使用 RxJS 操作符操作流的可能性给我们提供了许多能力。让我们考虑另一个常见用例。想象一下，你想要处理路由请求的 **超时**。当你的端点在一段时间后没有返回任何内容时，你想要以错误响应终止。以下构造使这成为可能：

```typescript
@@filename(timeout.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
@@switch
import { Injectable, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor {
  intercept(context, next) {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
```

5秒后，请求处理将被取消。你还可以添加自定义逻辑，在抛出 `RequestTimeoutException` 之前（例如释放资源）。