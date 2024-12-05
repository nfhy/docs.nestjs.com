### 模块

模块是一个用 `@Module()` 装饰器标注的类。`@Module()` 装饰器提供了 Nest 用来组织应用结构的元数据。

<figure><img class="illustrative-image" src="/assets/Modules_1.png" /></figure>

每个应用至少有一个模块，一个**根模块**。根模块是 Nest 用来构建**应用图**的起点——Nest 用来解析模块和提供者关系以及依赖的内部数据结构。虽然理论上非常小的应用可能只有根模块，但这不是典型情况。我们想强调的是，作为组织组件的有效方式，强烈推荐使用模块。因此，对于大多数应用来说，结果架构将使用多个模块，每个模块封装一组密切相关的**功能**。

`@Module()` 装饰器接受一个对象，其属性描述了模块：

|               |                                                                                                                                                                                                          |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `providers`   | 将由 Nest 注入器实例化的提供者，并且至少可以在此模块中共享                                                                                                                               |
| `controllers` | 在此模块中定义的一组控制器，必须被实例化                                                                                                                               |
| `imports`     | 导入的模块列表，这些模块导出了此模块所需的提供者                                                                                                                       |
| `exports`     | `providers` 的子集，由本模块提供，并且应该在导入此模块的其他模块中可用。你可以使用提供者本身或者只是它的令牌（`provide` 值） |

模块默认**封装**提供者。这意味着不可能注入既不直接是当前模块的一部分也没有从导入模块中导出的提供者。因此，你可以考虑模块导出的提供者作为模块的公共接口，或 API。

#### 功能模块

`CatsController` 和 `CatsService` 属于同一个应用领域。由于它们密切相关，将它们移动到一个功能模块中是有意义的。功能模块简单地组织了与特定功能相关的代码，保持代码组织并建立清晰的界限。这有助于我们管理复杂性，并随着应用和/或团队规模的增长，按照 [SOLID](https://en.wikipedia.org/wiki/SOLID) 原则进行开发。

为了演示这一点，我们将创建 `CatsModule`。

```typescript
@@filename(cats/cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

> 信息提示：要使用 CLI 创建模块，只需执行 `$ nest g module cats` 命令。

上述，我们在 `cats.module.ts` 文件中定义了 `CatsModule`，并将与此模块相关的所有内容移动到 `cats` 目录中。我们需要做的最后一件事是将此模块导入到根模块（`AppModule`，定义在 `app.module.ts` 文件中）。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

现在我们的目录结构如下：

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
      <div class="item">cats.module.ts</div>
      <div class="item">cats.service.ts</div>
    </div>
    <div class="item">app.module.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

#### 共享模块

在 Nest 中，模块默认是**单例**的，因此你可以轻松地在多个模块之间共享任何提供者的同一实例。

<figure><img class="illustrative-image" src="/assets/Shared_Module_1.png" /></figure>

每个模块自动成为**共享模块**。一旦创建，它可以被任何模块重用。假设我们想要在几个其他模块之间共享 `CatsService` 的实例。为了做到这一点，我们首先需要将 `CatsService` 提供者**导出**，通过将其添加到模块的 `exports` 数组中，如下所示：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

现在任何导入 `CatsModule` 的模块都可以访问 `CatsService`，并将与所有其他导入它的模块共享同一实例。

如果我们直接在每个需要它的模块中注册 `CatsService`，它确实可以工作，但这将导致每个模块获得自己的 `CatsService` 实例。这可能会导致内存使用增加，因为创建了同一服务的多个实例，并且如果服务维护任何内部状态，也可能导致意外行为，例如状态不一致。

通过将 `CatsService` 封装在模块中，例如 `CatsModule`，并导出它，我们确保所有导入 `CatsModule` 的模块重用相同的 `CatsService` 实例。这不仅减少了内存消耗，还导致了更可预测的行为，因为所有模块共享相同的实例，使得管理共享状态或资源更加容易。这是像 NestJS 这样的框架中模块化和依赖注入的关键好处之一——允许服务在整个应用中高效共享。

<app-banner-devtools></app-banner-devtools>

#### 模块重新导出

如上所述，模块可以导出其内部提供者。此外，它们可以重新导出它们导入的模块。在下面的示例中，`CommonModule` 既被导入也被导出自 `CoreModule`，使其对导入此模块的其他模块可用。

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

#### 依赖注入

模块类也可以**注入**提供者（例如，用于配置目的）：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
@@switch
import { Module, Dependencies } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
@Dependencies(CatsService)
export class CatsModule {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

然而，由于[循环依赖](/fundamentals/circular-dependency)，模块类本身不能被注入为提供者。

#### 全局模块

如果你必须在任何地方导入相同的模块集合，这可能会变得乏味。与 Nest 不同，[Angular](https://angular.dev) `providers` 在全局范围内注册。一旦定义，它们就随处可见。然而，Nest 将提供者封装在模块范围内。如果不先导入封装模块，你无法在其他地方使用模块的提供者。

当你想要提供一组应该在任何地方开箱即用的提供者（例如，帮助器、数据库连接等）时，使用 `@Global()` 装饰器使模块**全局**。

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

`@Global()` 装饰器使模块全局作用域。全局模块应该**只注册一次**，通常由根或核心模块注册。在上面的例子中，`CatsService` 提供者将无处不在，希望注入服务的模块不需要在它们的 imports 数组中导入 `CatsModule`。

> 信息提示：使一切都全局化不是一个好的设计决策。全局模块可用于减少必要的样板代码。`imports` 数组通常是使模块的 API 可供消费者使用的首选方式。

#### 动态模块

Nest 模块系统包括一个名为**动态模块**的强大功能。这个功能使你能够轻松创建可以动态注册和配置提供者的可定制模块。动态模块在[这里](/fundamentals/dynamic-modules)有详细说明。在这一章中，我们将提供一个简要的概览以完成对模块的介绍。

以下是一个 `DatabaseModule` 的动态模块定义示例：

```typescript
@@filename()
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
@@switch
import { Module } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options) {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> 信息提示：`forRoot()` 方法可以同步或异步返回动态模块（即，通过 `Promise`）。

这个模块默认定义了 `Connection` 提供者（在 `@Module()` 装饰器元数据中），但除此之外——根据传递给 `forRoot()` 方法的 `entities` 和 `options` 对象——还暴露了一系列提供者，例如仓库。请注意，动态模块返回的属性**扩展**（而不是覆盖）在 `@Module()` 装饰器中定义的基本模块元数据。这就是如何同时从模块导出静态声明的 `Connection` 提供者**和**动态生成的仓库提供者。

如果你想在全局范围内注册动态模块，将 `global` 属性设置为 `true`。

```typescript
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

> 警告：如上所述，使一切都全局化**不是一个好的设计决策**。

`DatabaseModule` 可以以以下方式导入和配置：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

如果你想反过来重新导出一个动态模块，可以在 exports 数组中省略 `forRoot()` 方法调用：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

[动态模块](/fundamentals/dynamic-modules)章节更详细地涵盖了这个话题，并包括了一个[工作示例](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)。

> 信息提示：了解如何使用 `ConfigurableModuleBuilder` 构建高度可定制的动态模块，[在这一章](/fundamentals/dynamic-modules#configurable-module-builder)。