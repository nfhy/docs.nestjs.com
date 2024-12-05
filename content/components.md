### 提供者

提供者是 Nest 中的一个基本概念。许多基本的 Nest 类可以被视为提供者——服务、存储库、工厂、助手等。提供者的主要思想是它可以被注入为依赖项；这意味着对象可以彼此创建各种关系，而“连接”这些对象的功能可以大部分委托给 Nest 运行时系统。

在上一章中，我们构建了一个简单的 `CatsController`。控制器应该处理 HTTP 请求并将更复杂的任务委托给提供者。提供者是普通的 JavaScript 类，在 NestJS 模块中被声明为 `providers`。更多信息，请参见“模块”章节。

> **提示** Nest 使得设计和组织依赖关系成为可能，我们强烈推荐遵循 [SOLID 原则](https://en.wikipedia.org/wiki/SOLID)。

#### 服务

我们先来创建一个简单的 `CatsService`。这个服务将负责数据存储和检索，并且被设计为由 `CatsController` 使用，因此它是一个很好的候选来被定义为提供者。

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  create(cat) {
    this.cats.push(cat);
  }

  findAll() {
    return this.cats;
  }
}
```

> **提示** 使用 CLI 创建服务，只需执行 `$ nest g service cats` 命令。

我们的 `CatsService` 是一个基本类，有一个属性和两个方法。唯一的新特性是它使用了 `@Injectable()` 装饰器。`@Injectable()` 装饰器附加了元数据，声明 `CatsService` 是一个可以由 Nest [IoC](https://en.wikipedia.org/wiki/Inversion_of_control) 容器管理的类。顺便说一句，这个例子还使用了一个 `Cat` 接口，它可能看起来像这样：

```typescript
@@filename(interfaces/cat.interface)
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

现在我们有了服务类来检索猫，让我们在 `CatsController` 中使用它：

```typescript
@@filename(cats.controller)
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Post, Body, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Post()
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

`CatsService` 通过类构造函数注入。注意使用 `private` 语法。这种简写允许我们在同一位置声明和初始化 `catsService` 成员。

#### 依赖注入

Nest 构建在通常被称为 **依赖注入** 的强大设计模式上。我们推荐阅读官方 [Angular 文档](https://angular.dev/guide/di) 中关于这个概念的优秀文章。

在 Nest 中，由于 TypeScript 的能力，管理依赖变得非常容易，因为它们是通过类型解析的。在下面的示例中，Nest 将通过创建并返回 `CatsService` 的实例（或者在正常情况下，如果它已经在其他地方被请求过，则返回现有实例）来解析 `catsService`。这个依赖被解析并传递到你的控制器的构造函数（或分配给指定的属性）：

```typescript
constructor(private catsService: CatsService) {}
```

#### 作用域

提供者通常有一个与应用程序生命周期同步的生命周期（“作用域”）。当应用程序启动时，必须解析每个依赖项，因此每个提供者都必须被实例化。类似地，当应用程序关闭时，每个提供者将被销毁。然而，也有方法可以使你的提供者生命周期 **请求作用域**。你可以在 [注入作用域](/fundamentals/injection-scopes) 章节中阅读更多关于这些技术的信息。

#### 自定义提供者

Nest 有一个内置的控制反转（“IoC”）容器，它解析提供者之间的关系。这个特性是上述描述的依赖注入特性的基础，但实际上比我们迄今为止描述的要强大得多。有几种方法可以定义提供者：你可以使用普通值、类，以及异步或同步工厂。在 [依赖注入](/fundamentals/dependency-injection) 章节中可以找到更多定义提供者的例子。

#### 可选提供者

有时，你可能有不一定需要解析的依赖项。例如，你的类可能依赖于一个 **配置对象**，但如果没有传递，应该使用默认值。在这种情况下，依赖项成为可选的，因为缺少配置提供者不会导致错误。

要指示提供者是可选的，请在构造函数的签名中使用 `@Optional()` 装饰器。

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

注意，在上述示例中我们使用了自定义提供者，这就是为什么我们包含了 `HTTP_OPTIONS` 自定义 **标记**。以前的示例显示了通过构造函数中的类指示依赖项的基于构造函数的注入。你可以在 [自定义提供者](/fundamentals/custom-providers) 章节中阅读更多关于自定义提供者及其关联标记的信息。

#### 基于属性的注入

到目前为止我们使用的技术被称为基于构造函数的注入，因为提供者是通过构造函数方法注入的。在某些特定情况下，**基于属性的注入** 可能很有用。例如，如果你的顶层类依赖于一个或多个提供者，通过在子类的构造函数中调用 `super()` 来传递它们可能会非常繁琐。为了避免这种情况，你可以在属性级别使用 `@Inject()` 装饰器。

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

> **警告** 如果你的类不继承另一个类，你应该始终优先使用 **基于构造函数的** 注入。构造函数明确列出了所需的依赖项，并比用 `@Inject` 注释的类属性提供更好的可见性。

#### 提供者注册

现在我们已经定义了一个提供者（`CatsService`），并且我们有一个服务的消费者（`CatsController`），我们需要将服务注册到 Nest，以便它可以执行注入。我们通过编辑我们的模块文件（`app.module.ts`）并将服务添加到 `@Module()` 装饰器的 `providers` 数组中来实现这一点。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

Nest 现在可以解析 `CatsController` 类的依赖项。

这就是我们的目录结构现在应该看起来的样子：

<div class="file-tree">
  <div class="item">src</div>
  <div class="children">
    <div class="item">cats</div>
    <div class="children">
      <div class="item">dto</div>
      <div class="children">
        <div class="item">create-cat.dto.ts</div>
      </div>
      <div class="item">interfaces</div>
      <div class="children">
        <div class="item">cat.interface.ts</div>
      </div>
      <div class="item">cats.controller.ts</div>
      <div class="item">cats.service.ts</div>
    </div>
    <div class="item">app.module.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

#### 手动实例化

到目前为止，我们已经讨论了 Nest 如何自动处理大部分依赖解析的细节。在某些情况下，你可能需要走出内置的依赖注入系统，手动检索或实例化提供者。我们下面简要讨论这两个主题。

要获取现有实例或动态实例化提供者，你可以使用 [模块引用](https://docs.nestjs.com/fundamentals/module-ref)。

要在 `bootstrap()` 函数中获取提供者（例如，对于没有控制器的独立应用程序，或在启动期间使用配置服务），请参阅 [独立应用程序](https://docs.nestjs.com/standalone-applications)。