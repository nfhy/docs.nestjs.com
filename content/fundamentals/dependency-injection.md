### 自定义提供者

在前面的章节中，我们讨论了 **依赖注入 (DI)** 在 Nest 中的各种方面以及它的使用方法。其中一个例子是 [基于构造函数的](https://docs.nestjs.com/providers#dependency-injection) 依赖注入，它用于将实例（通常是服务提供者）注入到类中。你不会惊讶地发现，依赖注入在 Nest 核心中以一种基本的方式构建。到目前为止，我们只探索了一种主要模式。随着你的应用程序变得更加复杂，你可能需要利用 DI 系统的全部特性，因此让我们更详细地探索它们。

#### DI 基础

依赖注入是一种 [控制反转 (IoC)](https://en.wikipedia.org/wiki/Inversion_of_control) 技术，其中你将依赖项的实例化委托给 IoC 容器（在我们的例子中，是 NestJS 运行时系统），而不是在你的代码中命令式地执行。让我们检查一下这个来自 [提供者章节](https://docs.nestjs.com/providers) 的例子中发生了什么。

首先，我们定义一个提供者。`@Injectable()` 装饰器将 `CatsService` 类标记为一个可以由 Nest IoC 容器管理的提供者。

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

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

  findAll() {
    return this.cats;
  }
}
```

然后我们请求 Nest 将提供者注入到我们的控制器类中：

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

最后，我们使用 Nest IoC 容器注册提供者：

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

到底在幕后发生了什么才能使这个工作？过程中有三个关键步骤：

1. 在 `cats.service.ts` 中，`@Injectable()` 装饰器声明 `CatsService` 类为一个可以由 Nest IoC 容器管理的类。
2. 在 `cats.controller.ts` 中，`CatsController` 通过构造函数注入声明对 `CatsService` 标记的依赖：
```typescript
  constructor(private catsService: CatsService)
```

3. 在 `app.module.ts` 中，我们将标记 `CatsService` 与 `cats.service.ts` 文件中的 `CatsService` 类关联起来。我们将在下面<a href="/fundamentals/custom-providers#standard-providers">看到</a>这种关联（也称为 _注册_）是如何发生的。

当 Nest IoC 容器实例化一个 `CatsController` 时，它首先寻找任何依赖项。当它找到 `CatsService` 依赖项时，它执行对 `CatsService` 标记的查找，根据注册步骤返回 `CatsService` 类。假设 `SINGLETON` 作用域（默认行为），Nest 将创建一个 `CatsService` 的实例，缓存它并返回它，或者如果已经缓存了一个，返回现有实例。

*这个解释有点简化，以说明要点。我们略过的一个重要领域是分析代码依赖项的过程非常复杂，并且发生在应用程序启动时。一个关键特性是依赖分析（或 "创建依赖图"）是 **传递性的**。在上面的例子中，如果 `CatsService` 本身有依赖项，这些依赖项也将被解析。依赖图确保依赖项以正确的顺序解析 - 本质上是 "自下而上"。这种机制使开发人员不必管理如此复杂的依赖图。*

<app-banner-courses></app-banner-courses>

#### 标准提供者

让我们更仔细地看看 `@Module()` 装饰器。在 `app.module` 中，我们声明：

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

`providers` 属性接受一个 `providers` 数组。到目前为止，我们已经通过类名列表提供了这些提供者。实际上，语法 `providers: [CatsService]` 是更完整语法的简写：

```typescript
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

现在我们看到了这个显式构造，我们可以了解注册过程。在这里，我们明确地将标记 `CatsService` 与类 `CatsService` 关联起来。简写符号只是一种方便，用于简化最常用的用例，其中标记用于通过同名类请求实例。

#### 自定义提供者

当你的需求超出了《标准提供者》所提供的范围时会发生什么？这里有一些例子：

- 你想创建一个自定义实例，而不是让 Nest 实例化（或返回缓存实例）一个类
- 你想在第二个依赖项中重用一个现有类
- 你想为测试用例用模拟版本覆盖一个类

Nest 允许你定义自定义提供者来处理这些情况。它提供了几种定义自定义提供者的方法。让我们逐一了解它们。

> info **提示** 如果你在依赖项解析方面遇到问题，可以设置 `NEST_DEBUG` 环境变量，并在启动期间获得额外的依赖项解析日志。

#### 值提供者：`useValue`

`useValue` 语法适用于注入一个常量值，将外部库放入 Nest 容器中，或用模拟对象替换真实实现。假设你想让 Nest 在测试目的下使用模拟的 `CatsService`。

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  /* 模拟实现
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

在这个例子中，`CatsService` 标记将解析为 `mockCatsService` 模拟对象。`useValue` 需要一个值 - 在这种情况下是一个字面量对象，它与它替换的 `CatsService` 类具有相同的接口。由于 TypeScript 的 [结构化类型](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)，你可以使用任何具有兼容接口的对象，包括字面量对象或使用 `new` 实例化的类实例。

#### 非类基础提供者标记

到目前为止，我们已经使用类名作为我们的提供者标记（`providers` 数组中提供者列表中的 `provide` 属性的值）。这与 [基于构造函数注入](https://docs.nestjs.com/providers#dependency-injection) 的标准模式相匹配，其中标记也是类名。（如果你对这个概念不是完全清楚，可以参考上面的 <a href="/fundamentals/custom-providers#di-fundamentals">DI 基础</a> 来回顾标记）。有时，我们可能想要使用字符串或符号作为 DI 标记的灵活性。例如：

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

在这个例子中，我们将字符串值标记（`'CONNECTION'`）与我们从外部文件导入的现有 `connection` 对象关联起来。

> warning **注意** 除了使用字符串作为标记值外，你还可以使用 JavaScript [符号](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) 或 TypeScript [枚举](https://www.typescriptlang.org/docs/handbook/enums.html)。

我们之前已经看到了如何使用标准的 [基于构造函数注入](https://docs.nestjs.com/providers#dependency-injection) 模式注入提供者。这种模式 **要求** 依赖项用类名声明。`'CONNECTION'` 自定义提供者使用字符串值标记。让我们看看如何注入这样的提供者。为此，我们使用 `@Inject()` 装饰器。这个装饰器接受一个参数 - 标记。

```typescript
@@filename()
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
@@switch
@Injectable()
@Dependencies('CONNECTION')
export class CatsRepository {
  constructor(connection) {}
}
```

> info **提示** `@Inject()` 装饰器是从 `@nestjs/common` 包导入的。

虽然我们在上面的例子中直接使用字符串 `'CONNECTION'` 用于说明目的，但为了代码组织的清洁，最好的做法是在单独的文件中定义标记，例如 `constants.ts`。将它们视为在它们自己的文件中定义的符号或枚举，并在需要时导入它们。

#### 类提供者：`useClass`

`useClass` 语法允许你动态确定一个标记应该解析到的类。例如，我们有一个抽象的（或默认的）`ConfigService` 类。根据当前环境，我们希望 Nest 提供配置服务的不同实现。以下代码实现了这样的策略。

```typescript
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

让我们看看这个代码示例中的一些细节。你会注意到我们首先用字面量对象定义 `configServiceProvider`，然后将其传递到模块装饰器的 `providers` 属性中。这只是一点代码组织，但在功能上等同于我们在本章中迄今为止使用的例子。

另外，我们使用 `ConfigService` 类名作为我们的标记。对于任何依赖于 `ConfigService` 的类，Nest 将注入提供的类实例（`DevelopmentConfigService` 或 `ProductionConfigService`），覆盖任何其他地方声明的默认实现（例如，用 `@Injectable()` 装饰器声明的 `ConfigService`）。

#### 工厂提供者：`useFactory`

`useFactory` 语法允许动态创建提供者。实际提供者将由工厂函数返回的值提供。工厂函数可以简单或复杂，需要。一个简单的工厂可能不需要依赖任何其他提供者。一个更复杂的工厂可以自己注入它需要的其他提供者来计算结果。对于后一种情况，工厂提供者语法有一对相关机制：

1. 工厂函数可以接受（可选的）参数。
2. （可选的）`inject` 属性接受一个 Nest 将解析并在实例化过程中作为参数传递给工厂函数的提供者数组。此外，这些提供者可以被标记为可选的。两个列表应该是相关的：Nest 将按相同顺序从 `inject` 列表中传递实例作为参数给工厂函数。以下示例演示了这一点。

```typescript
@@filename()
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: MyOptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/             \__________________/
  //        这个提供者                可以解析为 `undefined` 的这个提供者
  //        是强制性的。                标记
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // 类基础提供者
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
@@switch
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider, optionalProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/            \__________________/
  //        这个提供者              可以解析为 `undefined` 的这个提供者
  //        是强制性的。              标记
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // 类基础提供者
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

#### 别名提供者：`useExisting`

`useExisting` 语法允许你为现有提供者创建别名。这创建了两种访问同一提供者的方式。在以下示例中，（基于字符串的）标记 `'AliasedLoggerService'` 是（基于类的）标记 `LoggerService` 的别名。假设我们有两个不同的依赖项，一个用于 `'AliasedLoggerService'`，一个用于 `LoggerService`。如果两个依赖项都使用 `SINGLETON` 作用域声明，它们都将解析为同一个实例。

```typescript
@Injectable()
class LoggerService {
  /* 实现细节 */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

#### 非服务基础提供者

虽然提供者通常提供服务，但它们不仅限于此用途。提供者可以提供 **任何** 值。例如，一个提供者可以根据当前环境提供基于数组的配置对象，如下所示：

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

#### 导出自定义提供者

像任何提供者一样，自定义提供者的作用域限制在其声明模块内。要使其对其他模块可见，必须导出它。要导出自定义提供者，可以使用其标记或完整的提供者对象。

以下示例显示了使用标记导出：

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

或者，使用完整的提供者对象导出：

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```