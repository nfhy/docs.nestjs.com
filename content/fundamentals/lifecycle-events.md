### 生命周期事件

Nest 应用程序以及每个应用程序元素的生命周期都由 Nest 管理。Nest 提供了**生命周期钩子**，这些钩子可以让您了解关键的生命周期事件，并在发生这些事件时采取行动（在模块、提供者或控制器上运行注册的代码）。

#### 生命周期序列

下面的图表描述了从应用程序启动到节点进程退出的关键应用程序生命周期事件的顺序。我们可以将整个生命周期分为三个阶段：**初始化**、**运行**和**终止**。利用这个生命周期，您可以计划适当地初始化模块和服务，管理活动连接，并在接收到终止信号时优雅地关闭应用程序。

<figure><img class="illustrative-image" src="/assets/lifecycle-events.png" /></figure>

#### 生命周期事件

生命周期事件在应用程序启动和关闭期间发生。Nest 在以下生命周期事件中调用模块、提供者和控制器上注册的生命周期钩子方法（**关闭钩子**需要首先启用，如下所述[下面](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown)）。如下所示，在图表中，Nest 还调用相应的底层方法开始监听连接，并停止监听连接。

在下表中，`onModuleInit` 和 `onApplicationBootstrap` 只有在您显式调用 `app.init()` 或 `app.listen()` 时才会触发。

在下表中，`onModuleDestroy`、`beforeApplicationShutdown` 和 `onApplicationShutdown` 只有在您显式调用 `app.close()` 或进程接收到特殊系统信号（例如 SIGTERM）并且您已正确调用应用程序启动时的 `enableShutdownHooks`（见下文**应用程序关闭**部分）时才会触发。

| 生命周期钩子方法           | 触发钩子方法调用的生命周期事件                                                                                                                                                                   |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onModuleInit()`                | 调用一次，当宿主模块的依赖项已解决时。                                                                                                                                                    |
| `onApplicationBootstrap()`      | 调用一次，所有模块都已初始化，但在监听连接之前。                                                                                                                              |
| `onModuleDestroy()`\*           | 在接收到终止信号（例如，`SIGTERM`）后调用。                                                                                                                                            |
| `beforeApplicationShutdown()`\* | 在所有 `onModuleDestroy()` 处理程序完成后调用（Promise 解决或拒绝）；<br />一旦完成（Promise 解决或拒绝），将关闭所有现有连接（调用 `app.close()`）。 |
| `onApplicationShutdown()`\*     | 在连接关闭后调用（`app.close()` 解决）。                                                                                                                                                          |

\* 对于这些事件，如果您没有显式调用 `app.close()`，则必须选择加入以使它们使用系统信号如 `SIGTERM` 工作。见[应用程序关闭](fundamentals/lifecycle-events#application-shutdown)部分。

> 警告 **警告** 上述列出的生命周期钩子不会为 **请求范围** 类触发。请求范围类不与应用程序生命周期绑定，它们的寿命是不可预测的。它们专门针对每个请求创建，并在响应发送后自动垃圾回收。

> 提示 **提示** `onModuleInit()` 和 `onApplicationBootstrap()` 的执行顺序直接取决于模块导入的顺序，等待前一个钩子。

#### 使用方法

每个生命周期钩子都由一个接口表示。接口在技术上是可选的，因为它们在 TypeScript 编译后不存在。尽管如此，使用它们是一个好习惯，以便从强类型和编辑器工具中受益。要注册一个生命周期钩子，实现适当的接口。例如，要在特定类（例如，控制器、提供者或模块）上注册一个在模块初始化期间被调用的方法，实现 `OnModuleInit` 接口，提供 `onModuleInit()` 方法，如下所示：

```typescript
@@filename()
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`模块已初始化。`);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  onModuleInit() {
    console.log(`模块已初始化。`);
  }
}
```

#### 异步初始化

`OnModuleInit` 和 `OnApplicationBootstrap` 钩子都允许您推迟应用程序初始化过程（返回一个 `Promise` 或将方法标记为 `async` 并在方法体中等待异步方法完成）。

```typescript
@@filename()
async onModuleInit(): Promise<void> {
  await this.fetch();
}
@@switch
async onModuleInit() {
  await this.fetch();
}
```

#### 应用程序关闭

`onModuleDestroy()`、`beforeApplicationShutdown()` 和 `onApplicationShutdown()` 钩子在终止阶段被调用（响应显式调用 `app.close()` 或接收系统信号如 SIGTERM 如果选择加入）。这个特性通常与 [Kubernetes](https://kubernetes.io/) 一起使用，用于管理容器的生命周期，由 [Heroku](https://www.heroku.com/) 用于 dynos 或类似服务。

关闭钩子监听器消耗系统资源，因此默认情况下它们是禁用的。要使用关闭钩子，您**必须启用监听器**，通过调用 `enableShutdownHooks()`：

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 开始监听关闭钩子
  app.enableShutdownHooks();

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> 警告 **警告** 由于平台固有的限制，NestJS 对 Windows 上的应用程序关闭钩子支持有限。您可以预期 `SIGINT` 会工作，以及 `SIGBREAK` 和在某种程度上 `SIGHUP` - [了解更多](https://nodejs.org/api/process.html#process_signal_events)。然而，`SIGTERM` 在 Windows 上永远不会工作，因为在任务管理器中杀死进程是无条件的，“即，没有办法让应用程序检测或阻止它”。这是一些来自 libuv 的 [相关文档](https://docs.libuv.org/en/v1.x/signal.html)，以了解更多关于如何在 Windows 上处理 `SIGINT`、`SIGBREAK` 等。另见 Node.js 文档的 [进程信号事件](https://nodejs.org/api/process.html#process_signal_events)。

> 信息 **信息** `enableShutdownHooks` 通过启动监听器消耗内存。在您在单个 Node 进程中运行多个 Nest 应用程序的情况下（例如，当使用 Jest 并行运行测试时），Node 可能会抱怨过多的监听器进程。因此，默认情况下不启用 `enableShutdownHooks`。当您在单个 Node 进程中运行多个实例时，请留意这种情况。

当应用程序接收到终止信号时，它将调用任何注册的 `onModuleDestroy()`、`beforeApplicationShutdown()`，然后是 `onApplicationShutdown()` 方法（如上所述的顺序），相应的信号作为第一个参数。如果注册的函数等待异步调用（返回一个 promise），Nest 将在 promise 解决或拒绝之前不会继续序列。

```typescript
@@filename()
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(signal); // 例如 "SIGINT"
  }
}
@@switch
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal) {
    console.log(signal); // 例如 "SIGINT"
  }
}
```

> 信息 **信息** 调用 `app.close()` 不会终止 Node 进程，但只会触发 `onModuleDestroy()` 和 `onApplicationShutdown()` 钩子，因此如果有某些间隔、长时间运行的后台任务等，进程不会自动终止。