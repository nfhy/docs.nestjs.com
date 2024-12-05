### 异常过滤器

HTTP [异常过滤器](/exception-filters)层与相应的微服务层之间的唯一区别在于，不应抛出`HttpException`，而应使用`RpcException`。

```typescript
throw new RpcException('无效的凭证。');
```

> 信息 **提示** `RpcException`类是从`@nestjs/microservices`包中导入的。

以上示例中，Nest将处理抛出的异常，并返回具有以下结构的`error`对象：

```json
{
  "status": "error",
  "message": "无效的凭证。"
}
```

#### 过滤器

微服务异常过滤器的行为与HTTP异常过滤器类似，有一个小区别。`catch()`方法必须返回一个`Observable`。

```typescript
@@filename(rpc-exception.filter)
import { Catch, RpcExceptionFilter, ArgumentsHost } from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { RpcException } from '@nestjs/microservices';

@Catch(RpcException)
export class ExceptionFilter implements RpcExceptionFilter<RpcException> {
  catch(exception: RpcException, host: ArgumentsHost): Observable<any> {
    return throwError(() => exception.getError());
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { throwError } from 'rxjs';

@Catch(RpcException)
export class ExceptionFilter {
  catch(exception, host) {
    return throwError(() => exception.getError());
  }
}
```

> 警告 **警告** 当使用[混合应用程序](/faq/hybrid-application)时，默认情况下不会启用全局微服务异常过滤器。

以下示例使用手动实例化的方法范围过滤器。就像基于HTTP的应用程序一样，您也可以使用控制器范围的过滤器（即，用`@UseFilters()`装饰器前缀控制器类）。

```typescript
@@filename()
@UseFilters(new ExceptionFilter())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseFilters(new ExceptionFilter())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```

#### 继承

通常，您会创建完全定制化的异常过滤器，以满足您的应用程序需求。然而，可能存在某些情况，您希望简单地扩展**核心异常过滤器**，并根据某些因素覆盖行为。

为了将异常处理委托给基底过滤器，您需要扩展`BaseExceptionFilter`并调用继承的`catch()`方法。

```typescript
@@filename()
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseRpcExceptionFilter } from '@nestjs/microservices';

@Catch()
export class AllExceptionsFilter extends BaseRpcExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    return super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseRpcExceptionFilter } from '@nestjs/microservices';

@Catch()
export class AllExceptionsFilter extends BaseRpcExceptionFilter {
  catch(exception, host) {
    return super.catch(exception, host);
  }
}
```

上述实现只是一个展示方法的示例。您扩展的异常过滤器的实现将包括您定制的**业务逻辑**（例如，处理各种条件）。