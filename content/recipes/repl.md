### 读取-评估-打印-循环 (REPL)

REPL 是一个简单的交互式环境，它接受单个用户输入，执行它们，并将结果返回给用户。
REPL 功能允许您直接从终端检查您的依赖图，并在提供者（和控制器）上直接调用方法。

#### 使用方法

要在 REPL 模式下运行您的 NestJS 应用程序，请创建一个新的 `repl.ts` 文件（与现有的 `main.ts` 文件一起）并添加以下代码：

```typescript
@@filename(repl)
import { repl } from '@nestjs/core';
import { AppModule } from './src/app.module';

async function bootstrap() {
  await repl(AppModule);
}
bootstrap();
@@switch
import { repl } from '@nestjs/core';
import { AppModule } from './src/app.module';

async function bootstrap() {
  await repl(AppModule);
}
bootstrap();
```

现在在您的终端中，使用以下命令启动 REPL：

```bash
$ npm run start -- --entryFile repl
```

> 信息 **提示** `repl` 返回一个 [Node.js REPL 服务器](https://nodejs.org/api/repl.html) 对象。

一旦启动并运行，您应该在控制台看到以下消息：

```bash
LOG [NestFactory] 正在启动 Nest 应用程序...
LOG [InstanceLoader] AppModule 依赖项已初始化
LOG REPL 已初始化
```

现在您可以开始与您的依赖图进行交互。例如，您可以检索 `AppService`（我们这里使用入门项目作为示例）并调用 `getHello()` 方法：

```typescript
> get(AppService).getHello()
'Hello World!'
```

您可以在终端中执行任何 JavaScript 代码，例如，将 `AppController` 的实例分配给局部变量，并使用 `await` 调用异步方法：

```typescript
> appController = get(AppController)
AppController { appService: AppService {} }
> await appController.getHello()
'Hello World!'
```

要显示给定提供者或控制器上所有可用的公共方法，请使用 `methods()` 函数，如下所示：

```typescript
> methods(AppController)

方法：
 ◻ getHello
```

要打印所有注册模块的列表以及它们的控制器和提供者，请使用 `debug()`。

```typescript
> debug()

AppModule:
 - 控制器：
  ◻ AppController
 - 提供者：
  ◻ AppService
```

快速演示：

<figure><img src="/assets/repl.gif" alt="REPL 示例" /></figure>

您可以在下面的部分中找到有关现有预定义的本地方法的更多信息。

#### 本地函数

内置的 NestJS REPL 提供了一些启动 REPL 时全局可用的本地函数。您可以调用 `help()` 列出它们。

如果您不记得函数的签名（即：预期的参数和返回类型），您可以调用 `<function_name>.help`。
例如：

```text
> $.help
检索注入或控制器的实例，否则抛出异常。
接口：$(token: InjectionToken) => any
```

> 信息 **提示** 这些函数接口是用 [TypeScript 函数类型表达式语法](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-type-expressions) 编写的。

| 函数     | 描述                                                                                                        | 签名                                                             |
| -------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `debug`  | 打印所有注册模块的列表以及它们的控制器和提供者。                                                              | `debug(moduleCls?: ClassRef \| string) => void`                       |
| `get` 或 `$` | 检索注入或控制器的实例，否则抛出异常。                                                                  | `get(token: InjectionToken) => any`                                   |
| `methods` | 显示给定提供者或控制器上所有可用的公共方法。                                                               | `methods(token: ClassRef \| string) => void`                          |
| `resolve` | 解析注入或控制器的瞬态或请求作用域实例，否则抛出异常。                                                    | `resolve(token: InjectionToken, contextId: any) => Promise<any>`      |
| `select`  | 允许通过模块树导航，例如，从选定的模块中提取特定实例。                                                  | `select(token: DynamicModule \| ClassRef) => INestApplicationContext` |

#### 监视模式

在开发过程中，运行 REPL 的监视模式以自动反映所有代码更改非常有用：

```bash
$ npm run start -- --watch --entryFile repl
```

这有一个缺点，REPL 的命令历史在每次重新加载后会被丢弃，这可能很麻烦。
幸运的是，有一个非常简单的解决方案。像这样修改您的 `bootstrap` 函数：

```typescript
async function bootstrap() {
  const replServer = await repl(AppModule);
  replServer.setupHistory(".nestjs_repl_history", (err) => {
    if (err) {
      console.error(err);
    }
  });
}
```

现在历史记录在运行/重新加载之间被保留。