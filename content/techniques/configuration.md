### 配置

应用程序通常在不同的**环境中**运行。根据不同的环境，应该使用不同的配置设置。例如，本地环境依赖于特定数据库凭据，这些凭据仅对本地数据库实例有效。生产环境将使用另一组数据库凭据。由于配置变量会变化，最佳实践是将[配置变量](https://12factor.net/config)存储在环境变量中。

在Node.js应用程序中，通过`process.env`全局变量可以访问外部定义的环境变量。我们可以尝试通过在每个环境中分别设置环境变量来解决多环境问题。这很快就会变得难以管理，特别是在开发和测试环境中，这些值需要容易被模拟和/或更改。

在Node.js应用程序中，常见的做法是使用`.env`文件，其中包含键值对，每个键代表一个特定的值，以表示每个环境。然后在不同的环境之间运行应用程序就只是交换正确的`.env`文件的问题。

在Nest中使用这种技术的一个好方法是创建一个`ConfigModule`，它提供了一个`ConfigService`，用于加载相应的`.env`文件。虽然你可以选择自己编写这样的模块，但为了方便，Nest提供了`@nestjs/config`包现成的。我们将在当前章节中介绍这个包。

#### 安装

要开始使用它，我们首先安装所需的依赖项。

```bash
$ npm i --save @nestjs/config
```

> 信息提示 **Hint** `@nestjs/config`包内部使用[dotenv](https://github.com/motdotla/dotenv)。

> 注意 **Note** `@nestjs/config`需要TypeScript 4.1或更高版本。

#### 开始使用

安装过程完成后，我们可以导入`ConfigModule`。通常，我们会将其导入到根`AppModule`中，并使用`.forRoot()`静态方法控制其行为。在此步骤中，解析并解析环境变量键/值对。稍后，我们将看到几种访问其他特性模块中`ConfigModule`的`ConfigService`类的选项。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
})
export class AppModule {}
```

上述代码将从默认位置（项目根目录）加载并解析`.env`文件，将`.env`文件中的键/值对与分配给`process.env`的环境变量合并，并将结果存储在一个私有结构中，你可以通过`ConfigService`访问。`forRoot()`方法注册了`ConfigService`提供者，该提供者提供了一个`get()`方法用于读取这些解析/合并的配置变量。由于`@nestjs/config`依赖于[dotenv](https://github.com/motdotla/dotenv)，它使用该包的规则来解决环境变量名称冲突。当一个键同时存在于运行时环境作为环境变量（例如，通过操作系统shell导出`export DATABASE_USER=test`）和`.env`文件中时，运行时环境变量优先。

一个示例`.env`文件如下所示：

```json
DATABASE_USER=test
DATABASE_PASSWORD=test
```

#### 自定义环境文件路径

默认情况下，该包在应用程序的根目录中查找`.env`文件。要指定另一个路径的`.env`文件，请设置传递给`forRoot()`的（可选）选项对象的`envFilePath`属性，如下所示：

```typescript
ConfigModule.forRoot({
  envFilePath: '.development.env',
});
```

您还可以像这样指定多个`.env`文件的路径：

```typescript
ConfigModule.forRoot({
  envFilePath: ['.env.development.local', '.env.development'],
});
```

如果在多个文件中找到变量，则第一个优先。

#### 禁用环境变量加载

如果您不想加载`.env`文件，而是想简单地访问运行时环境（如操作系统shell导出`export DATABASE_USER=test`）中的环境变量，请将选项对象的`ignoreEnvFile`属性设置为`true`，如下所示：

```typescript
ConfigModule.forRoot({
  ignoreEnvFile: true,
});
```

#### 全局使用模块

当您想在其他模块中使用`ConfigModule`时，您需要导入它（像使用任何Nest模块一样）。或者，通过将选项对象的`isGlobal`属性设置为`true`，声明其为[全局模块](https://docs.nestjs.com/modules#global-modules)，如下所示。在这种情况下，一旦在根模块（例如`AppModule`）中加载了它，您就不需要在其他模块中导入`ConfigModule`。

```typescript
ConfigModule.forRoot({
  isGlobal: true,
});
```

#### 自定义配置文件

对于更复杂的项目，您可能会使用自定义配置文件来返回嵌套配置对象。这允许您按功能（例如，与数据库相关的设置）对相关配置设置进行分组，并将相关设置存储在单独的文件中以独立管理它们。

自定义配置文件导出一个工厂函数，该函数返回配置对象。配置对象可以是任何任意嵌套的纯JavaScript对象。`process.env`对象将包含完全解析的环境变量键/值对（如上所述解析并合并了`.env`文件和外部定义的变量）。由于您控制返回的配置对象，因此可以添加任何所需的逻辑来转换值类型，设置默认值等。例如：

```typescript
@@filename(config/configuration)
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432
  }
});
```

我们使用传递给`ConfigModule.forRoot()`方法的选项对象的`load`属性来加载此文件：

```typescript
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
    }),
  ],
})
export class AppModule {}
```

> 注意 **Notice** 赋值给`load`属性的值是一个数组，允许您加载多个配置文件（例如`load: [databaseConfig, authConfig]`）。

使用自定义配置文件时，我们还可以管理自定义文件，如YAML文件。以下是一个使用YAML格式的配置示例：

```yaml
http:
  host: 'localhost'
  port: 8080

db:
  postgres:
    url: 'localhost'
    port: 5432
    database: 'yaml-db'

  sqlite:
    database: 'sqlite.db'
```

要读取和解析YAML文件，我们可以利用`js-yaml`包。

```bash
$ npm i js-yaml
$ npm i -D @types/js-yaml
```

安装包后，我们使用`yaml#load`函数来加载我们刚刚创建的YAML文件。

```typescript
@@filename(config/configuration)
import { readFileSync } from 'fs';
import * as yaml from 'js-yaml';
import { join } from 'path';

const YAML_CONFIG_FILENAME = 'config.yaml';

export default () => {
  return yaml.load(
    readFileSync(join(__dirname, YAML_CONFIG_FILENAME), 'utf8'),
  ) as Record<string, any>;
};
```

> 注意 **Note** Nest CLI在构建过程中不会自动将您的“资产”（非TS文件）移动到`dist`文件夹。要确保您的YAML文件被复制，您必须在`nest-cli.json`文件中的`compilerOptions#assets`对象中指定这一点。例如，如果`config`文件夹与`src`文件夹在同一级别，添加`compilerOptions#assets`，值为`"assets": [{"include": "../config/*.yaml", "outDir": "./dist/config"}]`。更多信息[在这里](/cli/monorepo#assets)。

只是一个快速说明 - 配置文件即使在使用NestJS的`ConfigModule`的`validationSchema`选项时也不会自动验证。如果您需要验证或想要应用任何转换，您将不得不在工厂函数内处理，在那里您可以完全控制配置对象。这使您可以根据需要实现任何自定义验证逻辑。

例如，如果您想确保端口在特定范围内，可以向工厂函数添加验证步骤：

```typescript
@@filename(config/configuration)
export default () => {
  const config = yaml.load(
    readFileSync(join(__dirname, YAML_CONFIG_FILENAME), 'utf8'),
  ) as Record<string, any>;

  if (config.http.port < 1024 || config.http.port > 49151) {
    throw new Error('HTTP port must be between 1024 and 49151');
  }

  return config;
};
```

现在，如果端口超出指定范围，应用程序将在启动时抛出错误。

#### 使用`ConfigService`

要从我们的`ConfigService`访问配置值，我们首先需要注入`ConfigService`。像任何提供者一样，我们需要导入其包含模块 - `ConfigModule` - 到将使用它的模块中（除非您将传递给`ConfigModule.forRoot()`方法的选项对象中的`isGlobal`属性设置为`true`）。像下面这样导入到一个特性模块中。

```typescript
@@filename(feature.module)
@Module({
  imports: [ConfigModule],
  // ...
})
```

然后我们可以使用标准构造函数注入它：

```typescript
constructor(private configService: ConfigService) {}
```

> 信息提示 **Hint** `ConfigService`从`@nestjs/config`包中导入。

并在我们的类中使用它：

```typescript
// 获取一个环境变量
const dbUser = this.configService.get<string>('DATABASE_USER');

// 获取一个自定义配置值
const dbHost = this.configService.get<string>('database.host');
```

如上所示，使用`configService.get()`方法通过传递变量名来获取一个简单的环境变量。您可以像上面所示（例如，`get<string>(...)`）进行TypeScript类型提示。`get()`方法还可以遍历一个嵌套的自定义配置对象（通过<a href="techniques/configuration#custom-configuration-files">自定义配置文件</a>创建），如第二个示例所示。

您也可以使用接口作为类型提示来获取整个嵌套的自定义配置对象：

```typescript
interface DatabaseConfig {
  host: string;
  port: number;
}

const dbConfig = this.configService.get<DatabaseConfig>('database');

// 现在可以使用`dbConfig.port`和`dbConfig.host`
const port = dbConfig.port;
```

`get()`方法还接受一个可选的第二个参数，定义一个默认值，当键不存在时将返回该值，如下所示：

```typescript
// 当"database.host"未定义时使用"localhost"
const dbHost = this.configService.get<string>('database.host', 'localhost');
```

`ConfigService`有两个可选的泛型（类型参数）。第一个是帮助防止访问不存在的配置属性。像下面这样使用它：

```typescript
interface EnvironmentVariables {
  PORT: number;
  TIMEOUT: string;
}

// 在代码中的某处
constructor(private configService: ConfigService<EnvironmentVariables>) {
  const port = this.configService.get('PORT', { infer: true });

  // TypeScript错误：这是无效的，因为URL属性在EnvironmentVariables中未定义
  const url = this.configService.get('URL', { infer: true });
}
```

通过将`infer`属性设置为`true`，`ConfigService#get`方法将根据接口自动推断属性类型，例如，`typeof port === "number"`（如果您不使用TypeScript的`strictNullChecks`标志），因为`PORT`在`EnvironmentVariables`接口中有`number`类型。

此外，使用`infer`功能，您甚至可以在使用点表示法时推断嵌套自定义配置对象的属性类型：

```typescript
constructor(private configService: ConfigService<{ database: { host: string } }>) {
  const dbHost = this.configService.get('database.host', { infer: true })!;
  // typeof dbHost === "string"                                          |
  //                                                                     +--> 非空断言运算符
}
```

第二个泛型依赖于第一个泛型，作为一个类型断言，以消除`ConfigService`的方法在`strictNullChecks`开启时可能返回的所有`undefined`类型。例如：

```typescript
// ...
constructor(private configService: ConfigService<{ PORT: number }, true>) {
  const port = this.configService.get('PORT', { infer: true });
  //    ^^^ 端口的类型将是'number'，因此您不再需要TS类型断言
}
```

#### 配置命名空间

`ConfigModule`允许您定义和加载多个自定义配置文件，如<a href="techniques/configuration#custom-configuration-files">自定义配置文件</a>所示。您可以使用嵌套配置对象管理复杂的配置对象层次结构，如该部分所示。或者，您可以使用`registerAs()`函数返回一个“命名空间化”的配置对象，如下所示：

```typescript
@@filename(config/database.config)
export default registerAs('database', () => ({
  host: process.env.DATABASE_HOST,
  port: process.env.DATABASE_PORT || 5432
}));
```

与自定义配置文件一样，在您的`registerAs()`工厂函数中，`process.env`对象将包含完全解析的环境变量键/值对（如上所述解析并合并了`.env`文件和外部定义的变量）。

> 信息提示 **Hint** `registerAs`函数从`@nestjs/config`包中导出。

使用`forRoot()`方法的选项对象的`load`属性加载命名空间配置，就像您加载自定义配置文件一样：

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}
```

现在，要从`database`命名空间获取`host`值，请使用点表示法。使用`'database'`作为前缀到属性名，对应于传递给`registerAs()`函数的第一个参数的命名空间名称：

```typescript
const dbHost = this.configService.get<string>('database.host');
```

一个合理的替代方案是直接注入`database`命名空间。这使我们能够从强类型中受益：

```typescript
constructor(
  @Inject(databaseConfig.KEY)
  private dbConfig: ConfigType<typeof databaseConfig>,
) {}
```

> 信息提示 **Hint** `ConfigType`从`@nestjs/config`包中导出。

#### 模块中的命名空间配置

要将命名空间配置用作应用程序中另一个模块的配置对象，您可以利用配置对象的`.asProvider()`方法。此方法将您的命名空间配置转换为提供者，然后可以将此提供者传递给您想要使用的模块的`forRootAsync`（或任何等效方法）。

示例如下：

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync(databaseConfig.asProvider()),
  ],
})
```

要了解`.asProvider()`方法的工作原理，让我们检查返回值：

```typescript
// .asProvider()方法的返回值
{
  imports: [ConfigModule.forFeature(databaseConfig)],
  useFactory: (configuration: ConfigType<typeof databaseConfig>) => configuration,
  inject: [databaseConfig.KEY]
}
```

这种结构允许您将命名空间配置无缝集成到您的模块中，确保您的应用程序保持组织和模块化，而无需编写样板代码，重复代码。

#### 缓存环境变量

由于访问`process.env`可能会很慢，您可以通过将传递给`ConfigModule.forRoot()`的选项对象的`cache`属性设置为增加`ConfigService#get`方法的性能，当涉及到存储在`process.env`中的变量时。

```typescript
ConfigModule.forRoot({
  cache: true,
});
```

#### 部分注册

到目前为止，我们已经在根模块（例如`AppModule`）中处理了配置文件，使用了`forRoot()`方法。也许您有一个更复杂的项目结构，具有特定于功能配置文件位于多个不同的目录中。而不是在根模块中加载所有这些文件，`@nestjs/config`包提供了一个名为**部分注册**的功能，它只引用与每个功能模块相关的配置文件。使用功能模块内的`forFeature()`静态方法执行此部分注册，如下所示：

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [ConfigModule.forFeature(databaseConfig)],
})
export class DatabaseModule {}
```

> 信息提示 **Warning** 在某些情况下，您可能需要使用`onModuleInit()`钩子而不是构造函数来访问通过部分注册加载的属性。这是因为`forFeature()`方法在模块初始化期间运行，而模块初始化的顺序是不确定的。如果您通过另一个模块，在构造函数中访问这种方式加载的值，那么配置所依赖的模块可能尚未初始化。`onModuleInit()`方法仅在所有依赖模块都已初始化后才运行，因此这种技术是安全的。

#### 模式验证

在应用程序启动期间抛出异常是标准做法，如果所需的环境变量未提供或不符合某些验证规则。`@nestjs/config`包支持两种不同的方法来实现这一点：

- [Joi](https://github.com/sideway/joi)内置验证器。使用Joi，您定义一个对象模式并验证JavaScript对象是否符合该模式。
- 自定义`validate()`函数，它接受环境变量作为输入。

要使用Joi，我们必须安装Joi包：

```bash
$ npm install --save joi
```

现在我们可以定义一个Joi验证模式，并通过`forRoot()`方法的选项对象的`validationSchema`属性传递它，如下所示：

```typescript
@@filename(app.module)
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().port().default(3000),
      }),
    }),
  ],
})
export class AppModule {}
```

默认情况下，所有模式键都被视为可选的。在这里，我们为`NODE_ENV`和`PORT`设置了默认值，如果我们在环境（`.env`文件或进程环境）中没有提供这些变量，将使用这些默认值。或者，我们可以使用`required()`验证方法要求值必须在环境（`.env`文件或进程环境）中定义。在这种情况下，如果我们没有在环境中提供变量，验证步骤将抛出异常。查看[Joi验证方法](https://joi.dev/api/?v=17.3.0#example)了解更多如何构建验证模式。

默认情况下，未知环境变量（环境变量的键不在模式中）是允许的，并且不会引发验证异常。默认情况下，所有验证错误都会报告。您可以通过传递一个选项对象通过`validationOptions`键的`forRoot()`选项对象来更改这些行为。此选项对象可以包含任何标准验证选项属性，由[Joi验证选项](https://joi.dev/api/?v=17.3.0#anyvalidatevalue-options)提供。例如，要反转上述两个设置，像这样传递选项：

```typescript
@@filename(app.module)
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().port().default(3000),
      }),
      validationOptions: {
        allowUnknown: false,
        abortEarly: true,
      },
    }),
  ],
})
export class AppModule {}
```

`@nestjs/config`包使用默认设置：

- `allowUnknown`：控制是否允许环境变量中出现未知键。默认为`true`
- `abortEarly`：如果为true，则在第一个错误时停止验证；如果为false，则返回所有错误。默认为`false`。

请注意，一旦您决定传递一个`validationOptions`对象，任何您没有明确传递的设置将默认为`Joi`标准默认值（而不是`@nestjs/config`默认值）。例如，如果您在自定义`validationOptions`对象中未指定`allowUnknowns`，它将具有`Joi`默认值`false`。因此，最安全的做法是指定**这两个**设置。

#### 自定义验证函数

或者，您可以指定一个**同步**`validate`函数，该函数接受包含环境变量（来自env文件和进程）的对象，并返回包含验证环境变量的对象，以便您可以在需要时进行转换/突变。如果该函数抛出错误，它将阻止应用程序启动。

在这个例子中，我们将使用`class-transformer`和`class-validator`包。首先，我们必须定义：

- 一个带有验证约束的类，
- 一个使用`plainToInstance`和`validateSync`函数的验证函数。

```typescript
@@filename(env.validation)
import { plainToInstance } from 'class-transformer';
import { IsEnum, IsNumber, Max, Min, validateSync } from 'class-validator';

enum Environment {
  Development = "development",
  Production = "production",
  Test = "test",
  Provision = "provision",
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  @Min(0)
  @Max(65535)
  PORT: number;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToInstance(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );
  const errors = validateSync(validatedConfig, { skipMissingProperties: false });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }
  return validatedConfig;
}
```

有了这个，使用`validate`函数作为`ConfigModule`的配置选项，如下所示：

```typescript
@@filename(app.module)
import { validate } from './env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      validate,
    }),
  ],
})
export class AppModule {}
```

#### 自定义获取器函数

`ConfigService`定义了一个泛型`get()`方法，通过键来检索配置值。我们也可以添加`getter`函数，以实现更自然的编码风格：

```typescript
@@filename()
@Injectable()
export class ApiConfigService {
  constructor(private configService: ConfigService) {}

  get isAuthEnabled(): boolean {
    return this.configService.get('AUTH_ENABLED') === 'true';
  }
}
@@switch
@Dependencies(ConfigService)
@Injectable()
export class ApiConfigService {
  constructor(configService) {
    this.configService = configService;
  }

  get isAuthEnabled() {
    return this.configService.get('AUTH_ENABLED') === 'true';
  }
}
```

现在我们可以使用getter函数如下：

```typescript
@@filename(app.service)
@Injectable()
export class AppService {
  constructor(apiConfigService: ApiConfigService) {
    if (apiConfigService.isAuthEnabled) {
      // 启用了身份验证
    }
  }
}
@@switch
@Dependencies(ApiConfigService)
@Injectable()
export class AppService {
  constructor(apiConfigService) {
    if (apiConfigService.isAuthEnabled) {
      // 启用了身份验证
    }
  }
}
```

#### 环境变量加载钩子

如果模块配置依赖于环境变量，而这些变量是从`.env`文件中加载的，您可以使用`ConfigModule.envVariablesLoaded`钩子来确保在与`process.env`对象交互之前文件已加载，如下例所示：

```typescript
export async function getStorageModule() {
  await ConfigModule.envVariablesLoaded;
  return process.env.STORAGE === 'S3' ? S3StorageModule : DefaultStorageModule;
}
```

这种构造确保在`ConfigModule.envVariablesLoaded` Promise解决后，所有配置变量都已加载。

#### 条件模块配置

有时您可能希望根据环境变量有条件地加载模块，并在环境变量中指定条件。幸运的是，`@nestjs/config`提供了一个`ConditionalModule`，允许您这样做。

```typescript
@Module({
  imports: [
    ConfigModule.forRoot(),
    ConditionalModule.registerWhen(FooModule, 'USE_FOO'),
  ],
})
export class AppModule {}
```

上述模块只有在`.env`文件中没有`USE_FOO`环境变量的`false`值时才会加载`FooModule`。您也可以传递自定义条件，一个接收`process.env`引用的函数，应返回布尔值供`ConditionalModule`处理：

```typescript
@Module({
  imports: [
    ConfigModule.forRoot(),
    ConditionalModule.registerWhen(
      FooBarModule,
      (env: NodeJS.ProcessEnv) => !!env['foo'] && !!env['bar'],
    ),
  ],
})
export class AppModule {}
```

使用`ConditionalModule`时，请确保`ConfigModule`也已加载到应用程序中，以便`ConfigModule.envVariablesLoaded`钩子可以被正确引用和利用。如果钩子在5秒内或用户在`registerWhen`方法的第三个选项参数中设置的超时毫秒数内没有翻转为true，则`ConditionalModule`将抛出错误，Nest将中止启动应用程序。

#### 可扩展变量

`@nestjs/config`包支持环境变量扩展。使用这种技术，您可以创建嵌套环境变量，其中一个变量在另一个变量的定义中被引用。例如：

```json
APP_URL=mywebsite.com
SUPPORT_EMAIL=support@${APP_URL}
```

通过这种构造，变量`SUPPORT_EMAIL`解析为`'support@mywebsite.com'`。注意使用`${{ '{' }}...{{ '}' }}`语法来触发变量`APP_URL`在`SUPPORT_EMAIL`定义中的值的解析。

> 信息提示 **Hint** 对于此功能，`@nestjs/config`包内部使用[dotenv-expand](https://github.com/motdotla/dotenv-expand)。

通过在传递给`ConfigModule`的`forRoot()`方法的选项对象中设置`expandVariables`属性，启用环境变量扩展，如下所示：

```typescript
@@filename(app.module)
@Module({
  imports: [
    ConfigModule.forRoot({
      // ...
      expandVariables: true,
    }),
  ],
})
export class AppModule {}
```

#### 在`main.ts`中使用

虽然我们的配置存储在一个服务中，但它仍然可以在`main.ts`文件中使用。通过这种方式，您可以使用它来存储应用程序端口或CORS主机等变量。

要访问它，您必须使用`app.get()`方法，然后是服务引用：

```typescript
const configService = app.get(ConfigService);
```

然后您可以像平常一样使用它，通过调用`get`方法并传递配置键：

```typescript
const port = configService.get('PORT');
```