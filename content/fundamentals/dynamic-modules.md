### 动态模块

[模块章节](/modules)涵盖了Nest模块的基础知识，并简要介绍了[动态模块](https://docs.nestjs.com/modules#dynamic-modules)。本章节将深入探讨动态模块的主题。完成本章后，您应该能够很好地理解它们是什么以及如何以及何时使用它们。

#### 引言

文档**概览**部分中的大多数应用代码示例都使用了常规的或静态的模块。模块定义了像[提供者](/providers)和[控制器](/controllers)这样的组件组，它们作为整体应用程序的一个模块化部分而组合在一起。它们为这些组件提供了一个执行上下文或作用域。例如，在模块中定义的提供者对于模块的其他成员是可见的，无需将它们导出。当一个提供者需要在模块外部可见时，它首先从其宿主模块中导出，然后导入到其消费模块中。

让我们通过一个熟悉的示例来了解。

首先，我们将定义一个`UsersModule`来提供并导出一个`UsersService`。`UsersModule`是`UsersService`的**宿主**模块。

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

接下来，我们将定义一个`AuthModule`，它导入`UsersModule`，使`UsersModule`的导出提供者在`AuthModule`内部可用：

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

这些构造允许我们在例如`AuthService`中注入`UsersService`，该服务托管在`AuthModule`中：

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  /*
    实现中使用this.usersService
  */
}
```

我们将此称为**静态**模块绑定。Nest需要连接模块的所有信息已经在宿主和消费模块中声明。让我们来解包在这个过程中发生了什么。Nest通过以下方式使`UsersService`在`AuthModule`中可用：

1. 实例化`UsersModule`，包括传递性导入`UsersModule`本身消费的其他模块，并传递性解析任何依赖项（参见[自定义提供者](https://docs.nestjs.com/fundamentals/custom-providers)）。
2. 实例化`AuthModule`，并使`UsersModule`的导出提供者对`AuthModule`中的组件可用（就像它们在`AuthModule`中声明的一样）。
3. 在`AuthService`中注入`UsersService`的实例。

#### 动态模块用例

使用静态模块绑定，消费模块没有机会**影响**宿主模块中的提供者是如何配置的。为什么这很重要？考虑我们有一个需要在不同用例中以不同方式行为的通用模块。这类似于许多系统中的“插件”概念，其中通用设施需要在被消费者使用之前进行一些配置。

Nest的一个好例子是**配置模块**。许多应用程序发现通过使用配置模块来外部化配置细节是有用的。这使得在不同的部署中动态更改应用程序设置变得容易：例如，开发人员的开发数据库，用于暂存/测试环境的暂存数据库等。通过将配置参数的管理委托给配置模块，应用程序源代码保持独立于配置参数。

挑战在于，配置模块本身，由于它是通用的（类似于“插件”），需要由其消费模块进行定制。这就是_动态模块_发挥作用的地方。使用动态模块功能，我们可以使我们的配置模块**动态**，以便消费模块可以使用API控制配置模块在导入时如何被定制。

换句话说，动态模块提供了一个API，用于将一个模块导入到另一个模块，并在导入时定制该模块的属性和行为，而不是像我们迄今为止所看到的静态绑定。

<app-banner-devtools></app-banner-devtools>

#### 配置模块示例

我们将使用[配置章节](https://docs.nestjs.com/techniques/configuration#service)中的示例代码的基本版本进行本节。本章结束时的完成版本可作为[此处](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)的工作示例。

我们的要求是使`ConfigModule`接受一个`options`对象来定制它。这是我们想要支持的功能。基本示例将`.env`文件的位置硬编码为项目根文件夹。假设我们想要使其可配置，以便您可以在任何您选择的文件夹中管理您的`.env`文件。例如，想象您想要在项目根下名为`config`的文件夹中存储各种`.env`文件（即`src`的兄弟文件夹）。您希望能够在不同项目中使用`ConfigModule`时选择不同的文件夹。

动态模块使我们能够将参数传递到被导入的模块中，以便我们可以更改其行为。让我们看看这是如何工作的。如果我们从消费模块的角度来看最终目标的样子，然后反向工作，这将有所帮助。首先，让我们快速回顾一下_静态_导入`ConfigModule`的示例（即，没有能力影响导入模块的行为）。请密切注意`@Module()`装饰器中的`imports`数组：

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

让我们考虑一个_动态模块_导入，我们正在传递一个配置对象，可能是什么样子的。比较这两个示例中`imports`数组之间的差异：

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

让我们看看上面的动态示例中发生了什么。什么是移动部分？

1. `ConfigModule`是一个普通类，所以我们可以推断它必须有一个名为`register()`的**静态方法**。我们知道它是静态的，因为我们是在`ConfigModule`类上调用它，而不是在类的**实例**上。注意：这个方法，我们很快将创建，可以有任何任意名称，但按照约定我们应该称之为`forRoot()`或`register()`。
2. `register()`方法由我们定义，所以我们可以像这样接受任何输入参数。在这种情况下，我们将接受一个简单的`options`对象，这是典型的情况。
3. 我们可以推断`register()`方法必须返回类似`module`的东西，因为它的返回值出现在熟悉的`imports`列表中，我们迄今为止看到它包括了一系列模块。

实际上，我们的`register()`方法将返回一个`DynamicModule`。动态模块只不过是在运行时创建的模块，与静态模块具有完全相同的属性，外加一个名为`module`的附加属性。让我们快速回顾一下静态模块声明的示例，密切注意传递到装饰器中的模块选项：

```typescript
@Module({
  imports: [DogsModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
```

动态模块必须返回一个具有完全相同接口的对象，外加一个名为`module`的附加属性。`module`属性作为模块的名称，应该与模块类的类名相同，如下例所示。

> 信息提示 对于动态模块，除了`module`之外，模块选项对象的所有属性都是可选的。

那么静态`register()`方法呢？我们现在可以看到它的工作是返回具有`DynamicModule`接口的对象。当我们调用它时，我们实际上是在为`imports`列表提供一个模块，类似于我们在静态情况下通过列出模块类名的方式。换句话说，动态模块API简单地返回一个模块，而不是在`@Module`装饰器中固定属性，我们以编程方式指定它们。

还有一些细节需要覆盖以帮助使画面完整：

1. 我们现在可以声明`@Module()`装饰器的`imports`属性不仅可以接受模块类名（例如，`imports: [UsersModule]`），还可以接受返回动态模块的函数（例如，`imports: [ConfigModule.register(...)]`）。
2. 动态模块本身可以导入其他模块。我们在这个示例中不会这样做，但如果动态模块依赖于其他模块中的提供者，您将使用可选的`imports`属性导入它们。同样，这与您使用`@Module()`装饰器声明静态模块的元数据的方式完全类似。

有了这些理解，我们现在可以看看我们的动态`ConfigModule`声明必须是什么样子。让我们尝试一下。

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}
```

现在应该清楚地看到各个部分是如何联系在一起的。调用`ConfigModule.register(...)`返回一个具有属性的`DynamicModule`对象，这些属性基本上与我们迄今为止通过`@Module()`装饰器提供的元数据相同。

> 信息提示 从`@nestjs/common`导入`DynamicModule`。

我们的动态模块目前还不太有趣，因为我们还没有引入我们所说的要配置的能力。让我们接下来解决这个问题。

#### 模块配置

自定义`ConfigModule`行为的明显解决方案是在静态`register()`方法中传递给它一个`options`对象，正如我们上面猜测的。让我们再次看看消费模块的`imports`属性：

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

这很好地处理了将`options`对象传递给我们的动态模块。然后我们如何在`ConfigModule`中使用那个`options`对象？让我们考虑一分钟。我们知道我们的`ConfigModule`基本上是一个提供和导出可注入服务的宿主-`ConfigService`-供其他提供者使用。实际上是我们的`ConfigService`需要读取`options`对象以自定义其行为。让我们暂时假设我们知道如何从`register()`方法中获取`options`并将其传递给`ConfigService`。有了这个假设，我们可以对服务进行一些更改，以根据`options`对象的属性自定义其行为。（**注意**：目前，由于我们实际上_没有_确定如何传递它，我们将硬编码`options`。我们马上会解决这个问题）。

```typescript
import { Injectable } from '@nestjs/common';
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

现在我们的`ConfigService`知道如何在我们指定的`options`文件夹中找到`.env`文件。

我们剩下的任务是设法将`options`对象从`register()`步骤注入到我们的`ConfigService`中。当然，我们将使用_依赖注入_来做到这一点。这是一个关键点，所以确保您理解它。我们的`ConfigModule`提供了`ConfigService`。`ConfigService`反过来依赖于只在运行时提供的`options`对象。所以，在运行时，我们需要首先将`options`对象绑定到Nest IoC容器中，然后让Nest将其注入到我们的`ConfigService`中。记住从**自定义提供者**章节，提供者可以[包括任何值](https://docs.nestjs.com/fundamentals/custom-providers#non-service-based-providers)不仅服务，所以我们可以使用依赖注入来处理一个简单的`options`对象。

让我们首先解决将选项对象绑定到IoC容器的问题。我们在静态`register()`方法中这样做。记住，我们正在动态构建一个模块，模块的一个属性是其提供者列表。所以我们需要做的是将我们的选项对象定义为一个提供者。这将使其可注入到`ConfigService`中，我们将在下一步中利用这一点。在下面的代码中，注意`providers`数组：

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}
```

现在我们可以完成这个过程，通过将`'CONFIG_OPTIONS'`提供者注入到`ConfigService`中。回想一下，当我们使用非类标记定义提供者时，我们需要使用`@Inject()`装饰器[如这里所述](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens)。

```typescript
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: Record<string, any>) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

最后一点：为了简单起见，我们上面使用了基于字符串的注入令牌（`'CONFIG_OPTIONS'`），但最佳实践是将其定义为一个常量（或`Symbol`）在单独的文件中，并导入该文件。例如：

```typescript
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```

#### 示例

本章的完整代码示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)找到。

#### 社区指南

您可能已经看到在一些`@nestjs/`包中使用了`forRoot`、`register`和`forFeature`等方法，并可能想知道所有这些方法之间的区别。没有硬性规则，但`@nestjs/`包尝试遵循这些指南：

当创建一个模块时：

- 使用`register`，您期望用特定配置配置一个动态模块，仅供调用模块使用。例如，Nest的`@nestjs/axios`：`HttpModule.register({ baseUrl: 'someUrl' })`。如果，在另一个模块中您使用`HttpModule.register({ baseUrl: 'somewhere else' })`，它将具有不同的配置。您可以为尽可能多的模块这样做。

- 使用`forRoot`，您期望一次性配置一个动态模块，并在多个地方重用该配置（尽管可能不知不觉地抽象化）。这就是为什么您有一个`GraphQLModule.forRoot()`，一个`TypeOrmModule.forRoot()`等。

- 使用`forFeature`，您期望使用动态模块的`forRoot`的配置，但需要修改特定于调用模块需求的某些配置（即这个模块应该可以访问哪个存储库，或者日志记录器应该使用的上下文）。

所有这些，通常，也有它们的异步对应物`registerAsync`、`forRootAsync`和`forFeatureAsync`，意味着相同的事情，但使用Nest的依赖注入进行配置。

#### 可配置模块构建器

由于手动创建高度可配置的动态模块，这些模块公开`async`方法（`registerAsync`、`forRootAsync`等）相当复杂，特别是对于新手，Nest公开了`ConfigurableModuleBuilder`类，该类促进了这一过程，并允许您仅用几行代码构建模块“蓝图”。

例如，让我们以上面使用的示例（`ConfigModule`）为例，并将其转换为使用`ConfigurableModuleBuilder`。首先，确保我们创建了一个专用接口，表示我们的`ConfigModule`接受的选项。

```typescript
export interface ConfigModuleOptions {
  folder: string;
}
```

有了这个，创建一个新的专用文件（与现有的`config.module.ts`文件一起）并将其命名为`config.module-definition.ts`。在该文件中，让我们利用`ConfigurableModuleBuilder`构建`ConfigModule`定义。

```typescript
@@filename(config.module-definition)
import { ConfigurableModuleBuilder } from '@nestjs/common';
import { ConfigModuleOptions } from './interfaces/config-module-options.interface';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
@@switch
import { ConfigurableModuleBuilder } from '@nestjs/common';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().build();
```

现在让我们打开`config.module.ts`文件并修改其实现以利用自动生成的`ConfigurableModuleClass`：

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {}
```

扩展`ConfigurableModuleClass`意味着`ConfigModule`现在不仅提供了`register`方法（如之前自定义实现），还提供了`registerAsync`方法，允许消费者异步配置该模块，例如，通过提供异步工厂：

```typescript
@Module({
  imports: [
    ConfigModule.register({ folder: './config' }),
    // 或者：
    
    // ConfigModule.registerAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     };
    //   },
    //   inject: [...任何额外依赖...]
    // }),
  ],
})
export class AppModule {}
```

最后，让我们更新`ConfigService`类以注入生成的模块选项提供者，而不是我们迄今为止使用的`'CONFIG_OPTIONS'`。

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) { ... }
}
```

#### 自定义方法键

`ConfigurableModuleClass`默认提供`register`及其对应`registerAsync`方法。要使用不同的方法名称，请使用`ConfigurableModuleBuilder#setClassMethodName`方法，如下所示：

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setClassMethodName('forRoot').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setClassMethodName('forRoot').build();
```

这种构造将指示`ConfigurableModuleBuilder`生成一个暴露`forRoot`和`forRootAsync`的类。示例：

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' }), // <-- 注意使用"forRoot"而不是"register"
    // 或者：
    
    // ConfigModule.forRootAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     };
    //   },
    //   inject: [...任何额外依赖...]
    // }),
  ],
})
export class AppModule {}
```

#### 自定义选项工厂类

由于`registerAsync`方法（或`forRootAsync`或任何其他名称，取决于配置）允许消费者传递一个提供者定义，该定义解析为模块配置，库消费者可能会潜在地提供一个类来用于构建配置对象。

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory,
    }),
  ],
})
export class AppModule {}
```

这个类，默认必须提供`create()`方法，该方法返回一个模块配置对象。然而，如果您的库遵循不同的命名约定，您可以更改该行为，并指示`ConfigurableModuleBuilder`期望一个不同的方法，例如`createConfigOptions`，使用`ConfigurableModuleBuilder#setFactoryMethodName`方法：

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setFactoryMethodName('createConfigOptions').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setFactoryMethodName('createConfigOptions').build();
```

现在，`ConfigModuleOptionsFactory`类必须暴露`createConfigOptions`方法（而不是`create`）：

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory, // <-- 这个类必须提供"createConfigOptions"方法
    }),
  ],
})
export class AppModule {}
```

#### 额外选项

有一些边缘情况，您的模块可能需要接收额外的选项，这些选项决定了它应该如何表现（一个很好的例子是这样的选项是`isGlobal`标志-或只是`global`），同时，这些选项不应该被包括在`MODULE_OPTIONS_TOKEN`提供者中（因为它们与注册在该模块中的服务/提供者无关，例如，`ConfigService`不需要知道其宿主模块是否注册为全局模块）。

在这种情况下，可以使用`ConfigurableModuleBuilder#setExtras`方法。参见以下示例：

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } = new ConfigurableModuleBuilder<ConfigModuleOptions>()
  .setExtras(
    {
      isGlobal: true,
    },
    (definition, extras) => ({
      ...definition,
      global: extras.isGlobal,
    }),
  )
  .build();
```

在上面的示例中，传递给`setExtras`方法的第一个参数是一个包含“额外”属性的默认值的对象。第二个参数是一个函数，它接受自动生成的模块定义（具有`provider`、`exports`等）和`extras`对象，该对象表示额外的属性（由消费者指定或默认）。这个函数返回的值是修改后的模块定义。在这个特定示例中，我们正在获取`extras.isGlobal`属性并将其分配给模块定义的`global`属性（这反过来决定了模块是否是全局的，更多[在这里](/modules#dynamic-modules)）。

现在当消费这个模块时，可以如下传递额外的`isGlobal`标志：

```typescript
@Module({
  imports: [
    ConfigModule.register({
      isGlobal: true,
      folder: './config',
    }),
  ],
})
export class AppModule {}
```

然而，由于`isGlobal`被声明为一个“额外”属性，它将不会在`MODULE_OPTIONS_TOKEN`提供者中可用：

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) {
    // "options"对象将不会有"isGlobal"属性
    // ...
  }
}
```

#### 扩展自动生成的方法

自动生成的静态方法（`register`、`registerAsync`等）如果需要，可以扩展，如下所示：

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass, ASYNC_OPTIONS_TYPE, OPTIONS_TYPE } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {
  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      // 这里自定义逻辑
      ...super.register(options),
    };
  }

  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      // 这里自定义逻辑
      ...super.registerAsync(options),
    };
  }
}
```

注意使用`OPTIONS_TYPE`和`ASYNC_OPTIONS_TYPE`类型，这些类型必须从模块定义文件中导出：

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE, ASYNC_OPTIONS_TYPE } = new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```