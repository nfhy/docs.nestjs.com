### Nest Commander

在[独立应用程序](/standalone-applications)文档的基础上，还有一个[nest-commander](https://jmcdo29.github.io/nest-commander)包，用于编写类似于典型Nest应用程序结构的命令行应用程序。

> 信息 **info** `nest-commander`是一个第三方包，并不由NestJS核心团队管理。请在[适当的仓库](https://github.com/jmcdo29/nest-commander/issues/new/choose)中报告与该库相关的问题。

#### 安装

就像任何其他包一样，你必须先安装它才能使用它。

```bash
$ npm i nest-commander
```

#### 命令文件

`nest-commander`通过`@Command()`装饰器为类和`@Option()`装饰器为类的方法提供了一种使用装饰器编写新的命令行应用程序的简便方法。每个命令文件都应该实现`CommandRunner`抽象类，并用`@Command()`装饰器装饰。

每个命令都被视为Nest中的`@Injectable()`，所以你的正常依赖注入仍然像预期的那样工作。唯一需要注意的是抽象类`CommandRunner`，每个命令都应该由它实现。`CommandRunner`抽象类确保所有命令都有一个返回`Promise<void>`并接受参数`string[], Record<string, any>`的`run`方法。`run`命令是你可以从这里启动所有逻辑的地方，它将接受任何没有匹配选项标志的参数，并作为一个数组传递进来，以防你真的需要处理多个参数。至于选项，`Record<string, any>`，这些属性的名称与给定`@Option()`装饰器的`name`属性相匹配，而它们的值与选项处理程序的返回值相匹配。如果你想要更好的类型安全性，你完全可以为你的选项创建一个接口。

#### 运行命令

类似于在NestJS应用程序中我们可以使用`NestFactory`为我们创建一个服务器，并使用`listen`运行它，`nest-commander`包提供了一个简单易用的API来运行你的服务器。导入`CommandFactory`并使用静态方法`run`并传入你的应用程序的根模块。这可能看起来像下面这样

```ts
import { CommandFactory } from 'nest-commander';
import { AppModule } from './app.module';

async function bootstrap() {
  await CommandFactory.run(AppModule);
}

bootstrap();
```

默认情况下，使用`CommandFactory`时Nest的日志记录器是禁用的。不过，可以提供它，作为`run`函数的第二个参数。你可以提供自定义的NestJS日志记录器，或者提供一个你想要保留的日志级别的数组 - 如果你只想打印Nest的错误日志，至少提供`['error']`可能会很有用。

```ts
import { CommandFactory } from 'nest-commander';
import { AppModule } from './app.module';
import { LogService } from './log.service';

async function bootstrap() {
  await CommandFactory.run(AppModule, new LogService());

  // 或者，如果你只想打印Nest的警告和错误
  await CommandFactory.run(AppModule, ['warn', 'error']);
}

bootstrap();
```

就是这样。在底层，`CommandFactory`会为你调用`NestFactory`并在必要时调用`app.close()`，所以你不需要担心内存泄漏。如果你需要添加一些错误处理，总是可以在`run`命令周围使用`try/catch`，或者你可以在`bootstrap()`调用上链式添加一些`.catch()`方法。

#### 测试

如果你不能轻松地测试一个超级棒的命令行脚本，那写它有什么用呢，对吧？幸运的是，`nest-commander`有一些工具，你可以利用这些工具完美地融入NestJS生态系统，对于任何Nestlings来说都会感到宾至如归。在测试模式下，你不是使用`CommandFactory`构建命令，而是可以使用`CommandTestFactory`并传入你的元数据，非常类似于`@nestjs/testing`中的`Test.createTestingModule`的工作方式。实际上，它在底层使用了这个包。你仍然可以在调用`compile()`之前链式添加`overrideProvider`方法，以便在测试中交换DI部件。

#### 汇总

以下类将等同于拥有一个CLI命令，可以接收子命令`basic`或直接调用，支持`-n`、`-s`和`-b`（以及它们的长标志）以及每个选项的自定义解析器。`--help`标志也得到了支持，这是指挥官的惯例。

```ts
import { Command, CommandRunner, Option } from 'nest-commander';
import { LogService } from './log.service';

interface BasicCommandOptions {
  string?: string;
  boolean?: boolean;
  number?: number;
}

@Command({ name: 'basic', description: 'A parameter parse' })
export class BasicCommand extends CommandRunner {
  constructor(private readonly logService: LogService) {
    super()
  }

  async run(
    passedParam: string[],
    options?: BasicCommandOptions,
  ): Promise<void> {
    if (options?.boolean !== undefined && options?.boolean !== null) {
      this.runWithBoolean(passedParam, options.boolean);
    } else if (options?.number) {
      this.runWithNumber(passedParam, options.number);
    } else if (options?.string) {
      this.runWithString(passedParam, options.string);
    } else {
      this.runWithNone(passedParam);
    }
  }

  @Option({
    flags: '-n, --number [number]',
    description: 'A basic number parser',
  })
  parseNumber(val: string): number {
    return Number(val);
  }

  @Option({
    flags: '-s, --string [string]',
    description: 'A string return',
  })
  parseString(val: string): string {
    return val;
  }

  @Option({
    flags: '-b, --boolean [boolean]',
    description: 'A boolean parser',
  })
  parseBoolean(val: string): boolean {
    return JSON.parse(val);
  }

  runWithString(param: string[], option: string): void {
    this.logService.log({ param, string: option });
  }

  runWithNumber(param: string[], option: number): void {
    this.logService.log({ param, number: option });
  }

  runWithBoolean(param: string[], option: boolean): void {
    this.logService.log({ param, boolean: option });
  }

  runWithNone(param: string[]): void {
    this.logService.log({ param });
  }
}

```

确保命令类被添加到一个模块中

```ts
@Module({
  providers: [LogService, BasicCommand],
})
export class AppModule {}
```

现在，要在`main.ts`中运行CLI，你可以这样做

```ts
async function bootstrap() {
  await CommandFactory.run(AppModule);
}

bootstrap();
```

就这样，你就有了一个命令行应用程序。

#### 更多信息

访问[nest-commander文档网站](https://jmcdo29.github.io/nest-commander)以获取更多信息、示例和API文档。