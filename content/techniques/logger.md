### 日志记录器

Nest 提供了一个内置的基于文本的日志记录器，用于应用程序启动和几个其他情况，例如显示捕获的异常（即系统日志记录）。此功能通过 `@nestjs/common` 包中的 `Logger` 类提供。你可以完全控制日志记录系统的行为，包括以下任何选项：

- 完全禁用日志记录
- 指定日志详细程度（例如，显示错误、警告、调试信息等）
- 覆盖默认日志记录器中的时间戳（例如，使用 ISO8601 标准作为日期格式）
- 完全覆盖默认日志记录器
- 通过扩展它来自定义默认日志记录器
- 利用依赖注入来简化应用程序的组合和测试

你还可以利用内置的日志记录器，或创建你自己的自定义实现，来记录你自己的应用程序级别的事件和消息。

对于更高级的日志记录功能，你可以使用任何 Node.js 日志记录包，例如 [Winston](https://github.com/winstonjs/winston)，来实现一个完全自定义的、生产级别的日志记录系统。

#### 基本自定义

要禁用日志记录，将（可选的）Nest 应用程序选项对象中的 `logger` 属性设置为 `false`，该对象作为第二个参数传递给 `NestFactory.create()` 方法。

```typescript
const app = await NestFactory.create(AppModule, {
  logger: false,
});
await app.listen(process.env.PORT ?? 3000);
```

要启用特定的日志级别，将 `logger` 属性设置为一个字符串数组，指定要显示的日志级别，如下所示：

```typescript
const app = await NestFactory.create(AppModule, {
  logger: ['error', 'warn'],
});
await app.listen(process.env.PORT ?? 3000);
```

数组中的值可以是 `'log'`、`'fatal'`、`'error'`、`'warn'`、`'debug'` 和 `'verbose'` 的任意组合。

> **提示**：要禁用默认日志记录器消息中的颜色，将 `NO_COLOR` 环境变量设置为某个非空字符串。

#### 自定义实现

你可以通过将 `logger` 属性的值设置为一个实现 `LoggerService` 接口的对象，来提供 Nest 用于系统日志记录的自定义日志记录器实现。例如，你可以告诉 Nest 使用内置的全局 JavaScript `console` 对象（它实现了 `LoggerService` 接口），如下所示：

```typescript
const app = await NestFactory.create(AppModule, {
  logger: console,
});
await app.listen(process.env.PORT ?? 3000);
```

实现你自己的自定义日志记录器是直接的。简单地实现 `LoggerService` 接口的每个方法，如下所示。

```typescript
import { LoggerService, Injectable } from '@nestjs/common';

@Injectable()
export class MyLogger implements LoggerService {
  /**
   * 写入一个 'log' 级别的日志。
   */
  log(message: any, ...optionalParams: any[]) {}

  /**
   * 写入一个 'fatal' 级别的日志。
   */
  fatal(message: any, ...optionalParams: any[]) {}

  /**
   * 写入一个 'error' 级别的日志。
   */
  error(message: any, ...optionalParams: any[]) {}

  /**
   * 写入一个 'warn' 级别的日志。
   */
  warn(message: any, ...optionalParams: any[]) {}

  /**
   * 写入一个 'debug' 级别的日志。
   */
  debug?(message: any, ...optionalParams: any[]) {}

  /**
   * 写入一个 'verbose' 级别的日志。
   */
  verbose?(message: any, ...optionalParams: any[]) {}
}
```

然后你可以通过 Nest 应用程序选项对象的 `logger` 属性提供一个 `MyLogger` 的实例。

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new MyLogger(),
});
await app.listen(process.env.PORT ?? 3000);
```

这种技术虽然简单，但不利用依赖注入为 `MyLogger` 类。这可能会带来一些挑战，特别是在测试时，并限制了 `MyLogger` 的可重用性。有关更好的解决方案，请参见下面的<a href="techniques/logger#dependency-injection">依赖注入</a>部分。

#### 扩展内置日志记录器

而不是从头开始编写日志记录器，你可以通过扩展内置的 `ConsoleLogger` 类并覆盖默认实现的选定行为来满足你的需求。

```typescript
import { ConsoleLogger } from '@nestjs/common';

export class MyLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string) {
    // 在这里添加你定制的逻辑
    super.error(...arguments);
  }
}
```

你可以如下面的<a href="techniques/logger#using-the-logger-for-application-logging">使用日志记录器进行应用程序日志记录</a>部分所述，在你的功能模块中使用这样的扩展日志记录器。

你可以通过传递一个实例通过应用程序选项对象的 `logger` 属性来告诉 Nest 使用你的扩展日志记录器进行系统日志记录（如上文<a href="techniques/logger#custom-logger-implementation">自定义实现</a>部分所示），或者使用<a href="techniques/logger#dependency-injection">依赖注入</a>部分所示的技术。如果你这样做，你应该小心调用 `super`，如上面的示例代码所示，以便将特定的日志方法调用委托给父类（内置）类，以便 Nest 可以依赖它期望的内置功能。

> **注意**：在上面的例子中，我们将 `bufferLogs` 设置为 `true` 以确保所有日志将被缓冲，直到附加自定义日志记录器（这种情况下是 `MyLogger`）并且应用程序初始化过程要么完成要么失败。如果初始化过程失败，Nest 将回退到原始的 `ConsoleLogger` 以打印出任何报告的错误消息。另外，你可以将 `autoFlushLogs` 设置为 `false`（默认 `true`）以手动刷新日志（使用 `Logger.flush()` 方法）。

在这里我们使用 `NestApplication` 实例上的 `get()` 方法来检索 `MyLogger` 对象的单例实例。这种技术本质上是一种“注入”日志记录器实例以供 Nest 使用的方法。`app.get()` 调用检索 `MyLogger` 的单例实例，并依赖于该实例首先在另一个模块中被注入，如上所述。

你也可以在你的功能类中注入这个 `MyLogger` 提供者，从而确保 Nest 系统日志记录和应用程序日志记录之间一致的日志行为。有关更多信息，请参见<a href="techniques/logger#using-the-logger-for-application-logging">使用日志记录器进行应用程序日志记录</a>和<a href="techniques/logger#injecting-a-custom-logger">注入自定义日志记录器</a>。

#### 使用日志记录器进行应用程序日志记录

我们可以结合上述几种技术，以提供 Nest 系统日志记录和我们自己的应用程序事件/消息日志记录之间一致的行为和格式。

一个好的做法是在每个服务中实例化 `@nestjs/common` 中的 `Logger` 类。我们可以在 `Logger` 构造函数中提供我们的服务名称作为 `context` 参数，如下所示：

```typescript
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name);

  doSomething() {
    this.logger.log('正在做某事...');
  }
}
```

在默认的日志记录器实现中，`context` 被打印在方括号中，如下例中的 `NestFactory`：

```bash
[Nest] 19096   - 12/08/2019, 7:12:59 AM   [NestFactory] 正在启动 Nest 应用程序...
```

如果我们通过 `app.useLogger()` 提供自定义日志记录器，它实际上将被 Nest 内部使用。这意味着我们的代码保持实现不可知，而我们可以很容易地通过调用 `app.useLogger()` 将默认日志记录器替换为我们的自定义一个。

通过这种方式，如果我们按照上一节的步骤并调用 `app.useLogger(app.get(MyLogger))`，从 `MyService` 调用 `this.logger.log()` 将导致调用 `MyLogger` 实例的 `log` 方法。

这应该适用于大多数情况。但如果你需要更多的自定义（比如添加和调用自定义方法），请参见下一节。

#### 带时间戳的日志

要为每个记录的消息启用时间戳日志记录，你可以在创建日志记录器实例时使用可选的 `timestamp: true` 设置。

```typescript
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name, { timestamp: true });

  doSomething() {
    this.logger.log('在这里做某事带时间戳 -\u003e');
  }
}
```

这将产生以下格式的输出：

```bash
[Nest] 19096   - 04/19/2024, 7:12:59 AM   [MyService] 正在做某事带时间戳 +5ms
```

注意行尾的 `+5ms`。对于每个日志语句，计算与前一条消息的时间差，并在行尾显示。

#### 注入自定义日志记录器

首先，使用以下代码扩展内置日志记录器。我们为 `ConsoleLogger` 类提供 `scope` 选项作为配置元数据，指定一个[瞬态](/fundamentals/injection-scopes)范围，以确保我们在每个功能模块中有 `MyLogger` 的唯一实例。在这个例子中，我们没有扩展个别 `ConsoleLogger` 方法（如 `log()`、`warn()` 等），尽管你可以选择这样做。

```typescript
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class MyLogger extends ConsoleLogger {
  customLog() {
    this.log('请喂猫！');
  }
}
```

接下来，创建一个 `LoggerModule` 如下所示，并从该模块提供 `MyLogger`。

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

接下来，将 `LoggerModule` 导入到你的功能模块中。由于我们扩展了默认的 `Logger`，我们有使用 `setContext` 方法的便利。因此，我们可以开始使用上下文感知的自定义日志记录器，如下所示：

```typescript
import { Injectable } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  constructor(private myLogger: MyLogger) {
    // 由于瞬态范围，CatsService 有自己的 MyLogger 唯一实例，
    // 所以在这里设置上下文不会影响其他服务中的其他实例
    this.myLogger.setContext('CatsService');
  }

  findAll(): Cat[] {
    // 你可以调用所有默认方法
    this.myLogger.warn('即将返回猫！');
    // 以及你的自定义方法
    this.myLogger.customLog();
    return this.cats;
  }
}
```

最后，在 `main.ts` 文件中按照如下所示指示 Nest 使用自定义日志记录器的一个实例。当然在这个例子中，我们实际上并没有自定义日志记录器行为（通过扩展 `Logger` 方法如 `log()`、`warn()` 等），所以这一步实际上并不需要。但如果你在这些方法中添加了自定义逻辑并希望 Nest 使用相同的实现，那么这一步**将是必需的**。

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(new MyLogger());
await app.listen(process.env.PORT ?? 3000);
```

> **提示**：或者，而不是将 `bufferLogs` 设置为 `true`，你可以暂时用 `logger: false` 指令禁用日志记录器。请注意，如果你向 `NestFactory.create` 提供 `logger: false`，直到你调用 `useLogger` 之前，将不会记录任何内容，所以你可能会错过一些重要的初始化错误。如果你不介意你的一些初始消息将用默认日志记录器记录，你可以简单地省略 `logger: false` 选项。

#### 使用外部日志记录器

生产应用程序通常具有特定的日志记录要求，包括高级过滤、格式化和集中日志记录。Nest 的内置日志记录器用于监视 Nest 系统行为，也可以在开发期间用于功能模块中的基本格式化文本日志记录，但生产应用程序通常利用专门的日志记录模块，如 [Winston](https://github.com/winstonjs/winston)。与任何标准 Node.js 应用程序一样，你可以在 Nest 中充分利用此类模块。