### 中间件

中间件是一个在路由处理器**之前**被调用的函数。中间件函数可以访问[请求](https://expressjs.com/en/4x/api.html#req)和[响应](https://expressjs.com/en/4x/api.html#res)对象，以及应用请求-响应循环中的`next()`中间件函数。通常用一个名为`next`的变量来表示`next`中间件函数。

<figure><img class="illustrative-image" src="/assets/Middlewares_1.png" /></figure>

Nest中间件默认等同于[express](https://expressjs.com/en/guide/using-middleware.html)中间件。以下是官方express文档对中间件能力的描述：

<blockquote class="external">
  中间件函数可以执行以下任务：
  <ul>
    <li>执行任何代码。</li>
    <li>对请求和响应对象进行更改。</li>
    <li>结束请求-响应循环。</li>
    <li>调用堆栈中的下一个中间件函数。</li>
    <li>如果当前中间件函数没有结束请求-响应循环，它必须调用<code>next()</code>以将控制权传递给下一个中间件函数。否则，请求将被挂起。</li>
  </ul>
</blockquote>

你可以通过函数或带有`@Injectable()`装饰器的类来实现自定义Nest中间件。类应该实现`NestMiddleware`接口，而函数没有任何特殊要求。让我们首先通过类方法实现一个简单的中间件特性。

```typescript
@@filename(logger.middleware)
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('请求...');
    next();
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerMiddleware {
  use(req, res, next) {
    console.log('请求...');
    next();
  }
}
```

#### 依赖注入

Nest中间件完全支持依赖注入。就像提供者和控制器一样，它们能够**注入**在同一模块中可用的依赖项。通常，这是通过`constructor`完成的。

#### 应用中间件

`@Module()`装饰器中没有中间件的位置。相反，我们使用模块类的`configure()`方法来设置它们。包含中间件的模块必须实现`NestModule`接口。让我们在`AppModule`级别设置`LoggerMiddleware`。

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```

在上面的例子中，我们已经为`CatsController`中先前定义的`/cats`路由处理器设置了`LoggerMiddleware`。我们还可以进一步通过将包含路由`path`和请求`method`的对象传递给`forRoutes()`方法来限制中间件仅适用于特定请求方法。在下面的示例中，注意我们导入了`RequestMethod`枚举来引用所需的请求方法类型。

```typescript
@@filename(app.module)
import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
@@switch
import { Module, RequestMethod } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

> 提示 **提示** `configure()`方法可以异步化使用`async/await`（例如，你可以在`configure()`方法体内`await`异步操作的完成）。

> 警告 **警告** 当使用`express`适配器时，NestJS应用默认会注册`body-parser`包中的`json`和`urlencoded`。这意味着如果你想通过`MiddlewareConsumer`自定义该中间件，你需要在创建应用时将`bodyParser`标志设置为`false`。

#### 路由通配符

也支持基于模式的路由。例如，星号用作**通配符**，将匹配任意字符组合：

```typescript
forRoutes({
  path: 'ab*cd',
  method: RequestMethod.ALL,
});
```

`'ab*cd'`路由路径将匹配`abcd`、`ab_cd`、`abecd`等。字符`?`、`+`、`*`和`()`可以在路由路径中使用，并且是它们正则表达式对应物的子集。连字符（`-`）和点（`.`）在基于字符串的路径中被字面解释。

> 警告 **警告** `fastify`包使用的是`path-to-regexp`包的最新版本，该版本不再支持通配符星号`*`。相反，你必须使用参数（例如，`(.*)`、`:splat*`）。

#### 中间件消费者

`MiddlewareConsumer`是一个辅助类。它提供了几个内置方法来管理中间件。所有这些方法都可以简单地**链式**在[流畅风格](https://en.wikipedia.org/wiki/Fluent_interface)中。`forRoutes()`方法可以接收一个单独的字符串、多个字符串、一个`RouteInfo`对象、一个控制器类，甚至是多个控制器类。在大多数情况下，你可能只会传递一个由逗号分隔的**控制器**列表。下面是一个单一控制器的示例：

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
```

> 提示 **提示** `apply()`方法可以接收单个中间件，或者多个参数来指定<a href="/middleware#multiple-middleware">多个中间件</a>。

#### 排除路由

有时我们想要**排除**某些路由，使其不应用中间件。我们可以使用`exclude()`方法轻松排除某些路由。此方法可以接收一个单独的字符串、多个字符串，或一个`RouteInfo`对象，以识别要排除的路由，如下所示：

```typescript
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/(.*)',
  )
  .forRoutes(CatsController);
```

> 提示 **提示** `exclude()`方法支持使用[path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters)包的通配符参数。

通过上述示例，`LoggerMiddleware`将被绑定到`CatsController`中定义的所有路由**除外**传递给`exclude()`方法的三个路由。

#### 功能性中间件

我们一直在使用的`LoggerMiddleware`类非常简单。它没有成员，没有额外的方法，也没有依赖项。为什么我们不能只定义一个简单的函数而不是类呢？实际上，我们可以。这种类型的中间件被称为**功能性中间件**。让我们将日志记录中间件从基于类的转换为功能性中间件以说明差异：

```typescript
@@filename(logger.middleware)
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`请求...`);
  next();
};
@@switch
export function logger(req, res, next) {
  console.log(`请求...`);
  next();
};
```

并在`AppModule`中使用它：

```typescript
@@filename(app.module)
consumer
  .apply(logger)
  .forRoutes(CatsController);
```

> 提示 **提示** 任何时候你的中间件不需要依赖项，考虑使用更简单的**功能性中间件**替代方案。

#### 多个中间件

如上所述，为了按顺序绑定多个中间件，只需在`apply()`方法中提供一个逗号分隔的列表：

```typescript
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

#### 全局中间件

如果我们想要一次性将中间件绑定到每个注册的路由，我们可以使用`INestApplication`实例提供的`use()`方法：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(process.env.PORT ?? 3000);
```

> 提示 **提示** 在全局中间件中访问DI容器是不可能的。当使用`app.use()`时，你可以使用[功能性中间件](middleware#functional-middleware)。或者，你可以使用类中间件，并在`AppModule`（或任何其他模块）中使用`.forRoutes('*')`消费它。