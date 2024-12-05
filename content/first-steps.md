### 初识Nest

在这篇文章系列中，你将学习到Nest应用的**核心基础知识**。为了熟悉Nest应用的基本构建模块，我们将构建一个基本的CRUD应用，其功能在入门水平上覆盖了广泛的领域。

#### 语言

我们钟爱[TypeScript](https://www.typescriptlang.org/)，但最重要的是——我们热爱[Node.js](https://nodejs.org/en/)。这就是为什么Nest兼容TypeScript和纯JavaScript。Nest利用了最新的语言特性，因此要使用纯JavaScript，我们需要一个[Babel](https://babeljs.io/)编译器。

我们将主要使用TypeScript提供示例，但你可以随时**切换代码片段**为纯JavaScript语法（只需点击每个代码片段右上角的语言切换按钮）。

#### 先决条件

请确保你的操作系统上安装了[Node.js](https://nodejs.org)（版本>=16）。

#### 设置

使用[Nest CLI](/cli/overview)设置新项目非常简单。安装了[npm](https://www.npmjs.com/)后，你可以使用以下命令在操作系统的终端中创建一个新的Nest项目：

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> 信息提示：要使用TypeScript的[更严格](https://www.typescriptlang.org/tsconfig#strict)的特性集创建一个新项目，向`nest new`命令传递`--strict`标志。

将创建`project-name`目录，安装节点模块和其他一些样板文件，并创建并填充`src/`目录与几个核心文件。

<div class="file-tree">
  <div class="item">src</div>
  <div class="children">
    <div class="item">app.controller.spec.ts</div>
    <div class="item">app.controller.ts</div>
    <div class="item">app.module.ts</div>
    <div class="item">app.service.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

以下是这些核心文件的简要概述：

|                          |                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `app.controller.ts`      | 一个带有单个路由的基本控制器。                                                                             |
| `app.controller.spec.ts` | 控制器的单元测试。                                                                                  |
| `app.module.ts`          | 应用的根模块。                                                                                 |
| `app.service.ts`         | 一个带有单个方法的基本服务。                                                                               |
| `main.ts`                | 应用的入口文件，它使用核心函数`NestFactory`来创建Nest应用实例。 |

`main.ts`包括一个异步函数，将**启动**我们的应用程序：

```typescript
@@filename(main)

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

要创建Nest应用实例，我们使用核心`NestFactory`类。`NestFactory`公开了一些静态方法，允许创建应用实例。`create()`方法返回一个应用对象，该对象实现了`INestApplication`接口。这个对象提供了一组方法，将在接下来的章节中描述。在上面的`main.ts`示例中，我们简单地启动了我们的HTTP监听器，让应用等待传入的HTTP请求。

请注意，使用Nest CLI创建的项目脚手架创建了一个初始项目结构，鼓励开发者遵循将每个模块保持在其自己的专用目录中的约定。

> 信息提示：默认情况下，如果在创建应用程序时发生任何错误，你的应用程序将以代码`1`退出。如果你想让它抛出错误而不是退出，可以禁用`abortOnError`选项（例如，`NestFactory.create(AppModule, { abortOnError: false })`）。

<app-banner-courses></app-banner-courses>

#### 平台

Nest旨在成为一个平台无关的框架。平台独立性使得开发者可以创建可跨多种不同类型的应用程序使用的可重用逻辑部分。从技术上讲，一旦创建了适配器，Nest就能与任何Node HTTP框架一起工作。有两个HTTP平台是开箱即用的：[express](https://expressjs.com/)和[fastify](https://www.fastify.io)。你可以选择最适合你需求的平台。

|                    |                                                                                                                                                                                                                                                                                                                                    |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `platform-express` | [Express](https://expressjs.com/)是一个众所周知的最小化Web框架，适用于Node。它是一个经过实战测试、生产就绪的库，社区实现了大量资源。默认情况下使用`@nestjs/platform-express`包。许多用户使用Express就能得到很好的服务，无需采取任何行动来启用它。 |
| `platform-fastify` | [Fastify](https://www.fastify.io/)是一个高性能、低开销的框架，高度专注于提供最大的效率和速度。[点击这里](/techniques/performance)了解如何使用它。                                                                                                                                  |

无论使用哪个平台，它都会暴露其自己的应用接口。这些分别被视为`NestExpressApplication`和`NestFastifyApplication`。

当你像下面的示例那样向`NestFactory.create()`方法传递一个类型时，`app`对象将具有该特定平台独有的方法。注意，你**不需要**指定一个类型**除非你**实际上想要访问底层平台API。

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
```

#### 运行应用程序

安装过程完成后，你可以在操作系统的命令提示符下运行以下命令来启动应用程序，以监听传入的HTTP请求：

```bash
$ npm run start
```

> 信息提示：为了加快开发过程（加快20倍的构建速度），你可以使用[SWC构建器](/recipes/swc)，通过向`start`脚本传递`-b swc`标志，如下`npm run start -- -b swc`。

这个命令以在`src/main.ts`文件中定义的端口启动应用程序。一旦应用程序运行，打开你的浏览器并导航到`http://localhost:3000/`。你应该看到`Hello World!`消息。

要监视文件的更改，你可以运行以下命令来启动应用程序：

```bash
$ npm run start:dev
```

这个命令将监视你的文件，自动重新编译并重新加载服务器。

#### 代码检查和格式化

[CLI](/cli/overview)提供了最大的努力，以在大规模开发工作流程中搭建可靠的脚手架。因此，生成的Nest项目预装了代码**检查器**和**格式化器**（分别是[eslint](https://eslint.org/)和[prettier](https://prettier.io/)）。

> 信息提示：不确定格式化器与检查器的角色？[在这里](https://prettier.io/docs/en/comparison.html)学习差异。

为了确保最大的稳定性和可扩展性，我们使用了基础的[eslint](https://www.npmjs.com/package/eslint)和[prettier](https://www.npmjs.com/package/prettier) cli包。这种设置允许与官方扩展的整洁IDE集成。

对于IDE不相关的无头环境（持续集成、Git钩子等），Nest项目提供了现成的`npm`脚本。

```bash
# 使用eslint检查并自动修复
$ npm run lint

# 使用prettier格式化
$ npm run format
```