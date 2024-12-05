### 引言

Nest（NestJS）是一个用于构建高效、可扩展的[Node.js](https://nodejs.org/)服务器端应用程序的框架。它使用渐进式JavaScript，构建于并完全支持[TypeScript](http://www.typescriptlang.org/)（尽管如此，它仍然允许开发者使用纯JavaScript进行编码），并结合了面向对象编程（OOP）、函数式编程（FP）和函数响应式编程（FRP）的元素。

在底层，Nest利用了强大的HTTP服务器框架，如[Express](https://expressjs.com/)（默认）和可选配置的[Fastify](https://github.com/fastify/fastify)！

Nest在这些常见的Node.js框架（Express/Fastify）之上提供了一个抽象层，但同时也直接向开发者暴露了它们的API。这使得开发者可以自由地使用为底层平台提供的大量第三方模块。

#### 理念

近年来，得益于Node.js，JavaScript已经成为前端和后端应用程序的“通用语言”。这催生了许多优秀的项目，如[Angular](https://angular.dev/)、[React](https://github.com/facebook/react)和[Vue](https://github.com/vuejs/vue)，它们提高了开发者的生产力，并使得创建快速、可测试、可扩展的前端应用程序成为可能。然而，尽管存在许多出色的库、助手和工具用于Node（和服务器端JavaScript），但它们并没有有效地解决主要问题——**架构**。

Nest提供了一个开箱即用的应用程序架构，允许开发者和团队创建高度可测试、可扩展、松耦合且易于维护的应用程序。该架构深受Angular的启发。

#### 安装

要开始使用，你可以选择使用[Nest CLI](/cli/overview)搭建项目，或者[克隆一个起始项目](#alternatives)（两者将产生相同的结果）。

要使用Nest CLI搭建项目，请运行以下命令。这将创建一个新的项目目录，并用初始的核心Nest文件和支持模块填充该目录，为你的项目创建一个传统的基础结构。对于首次用户，建议使用**Nest CLI**创建新项目。我们将继续在[第一步](first-steps)中采用这种方法。

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> 信息 **提示** 要创建一个具有更严格特性集的新的TypeScript项目，请在`nest new`命令中传递`--strict`标志。

#### 替代方案

或者，使用**Git**安装TypeScript起始项目：

```bash
$ git clone https://github.com/nestjs/typescript-starter.git project
$ cd project
$ npm install
$ npm run start
```

> 信息 **提示** 如果你想克隆没有git历史的仓库，可以使用[degit](https://github.com/Rich-Harris/degit)。

打开浏览器，导航到[`http://localhost:3000/`](http://localhost:3000/)。

要安装JavaScript版本的起始项目，请在上述命令序列中使用`javascript-starter.git`。

你也可以从零开始安装核心和支持包来启动一个新项目。请注意，你需要自己设置项目样板文件。至少，你需要这些依赖项：`@nestjs/core`、`@nestjs/common`、`rxjs`和`reflect-metadata`。查看这篇短文，了解如何创建一个完整的项目：[5步创建一个最小化的NestJS应用程序！](https://dev.to/micalevisk/5-steps-to-create-a-bare-minimum-nestjs-app-from-scratch-5c3b)。