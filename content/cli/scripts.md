### Nest CLI 和脚本

本节提供了关于 `nest` 命令如何与编译器和脚本交互以帮助 DevOps 人员管理开发环境的额外背景信息。

Nest 应用程序是一个**标准**的 TypeScript 应用程序，需要先编译成 JavaScript 才能执行。完成编译步骤有多种方式，开发者/团队可以自由选择最适合他们的方式。考虑到这一点，Nest 提供了一套开箱即用的工具，旨在实现以下目标：

- 提供一个标准的构建/执行过程，通过命令行可用，并且具有合理的默认设置，能够“直接工作”。
- 确保构建/执行过程是**开放**的，因此开发者可以直接访问底层工具，使用原生特性和选项进行自定义。
- 保持一个完全标准的 TypeScript/Node.js 框架，以便整个编译/部署/执行管道可以由开发团队选择使用的任何外部工具管理。

这一目标是通过 `nest` 命令、本地安装的 TypeScript 编译器和 `package.json` 脚本的结合来实现的。我们下面描述这些技术如何协同工作。这应该可以帮助你理解构建/执行过程中的每一步发生了什么，以及如何根据需要自定义该行为。

#### nest 二进制文件

`nest` 命令是一个操作系统级别的二进制文件（即，从操作系统命令行运行）。此命令实际上涵盖了以下三个不同的领域，如下所述。我们建议通过 `package.json` 脚本自动提供的构建（`nest build`）和执行（`nest start`）子命令来运行（如果你希望通过克隆仓库而不是运行 `nest new` 来开始，请参阅 [typescript starter](https://github.com/nestjs/typescript-starter)）。

#### 构建

`nest build` 是标准 `tsc` 编译器或 [标准项目](https://docs.nestjs.com/cli/overview#project-structure) 的 `swc` 编译器，或使用 `ts-loader` 的 webpack 打包器（对于 [monorepos](https://docs.nestjs.com/cli/overview#project-structure)）。它除了处理 `tsconfig-paths` 之外，不会添加任何其他编译特性或步骤。它存在的原因是因为大多数开发者，特别是刚开始使用 Nest 的时候，不需要调整编译器选项（例如 `tsconfig.json` 文件），这有时可能会很棘手。

有关更多详细信息，请参见 [nest build](https://docs.nestjs.com/cli/usages#nest-build) 文档。

#### 执行

`nest start` 仅确保项目已构建（与 `nest build` 相同），然后以一种便携、简单的方式调用 `node` 命令来执行编译后的应用程序。与构建一样，你可以按需自定义此过程，无论是使用 `nest start` 命令及其选项，还是完全替换它。整个过程是一个标准的 TypeScript 应用程序构建和执行管道，你可以按需管理该过程。

有关更多详细信息，请参见 [nest start](https://docs.nestjs.com/cli/usages#nest-start) 文档。

#### 生成

`nest generate` 命令，顾名思义，生成新的 Nest 项目或其中的组件。

#### 包脚本

在操作系统命令级别运行 `nest` 命令需要 `nest` 二进制文件全局安装。这是 npm 的标准功能，不在 Nest 的直接控制之下。一个后果是全局安装的 `nest` 二进制文件**不是**作为项目依赖项在 `package.json` 中管理的。例如，两个不同的开发者可能正在运行两个不同版本的 `nest` 二进制文件。对此的标准解决方案是使用包脚本，以便你可以将构建和执行步骤中使用的工具视为开发依赖项。

当你运行 `nest new` 或克隆 [typescript starter](https://github.com/nestjs/typescript-starter) 时，Nest 会在新项目的 `package.json` 脚本中填充 `build` 和 `start` 等命令。它还安装了底层编译器工具（如 `typescript`）作为**开发依赖项**。

你可以使用以下命令运行构建和执行脚本：

```bash
$ npm run build
```

和

```bash
$ npm run start
```

这些命令使用 npm 的脚本运行功能来使用**本地安装**的 `nest` 二进制文件执行 `nest build` 或 `nest start`。通过使用这些内置的包脚本，你可以完全依赖管理 Nest CLI 命令。这意味着，通过遵循这种**推荐**的用法，你的组织的所有成员都可以确保运行相同版本的命令。

*这适用于 `build` 和 `start` 命令。`nest new` 和 `nest generate` 命令不是构建/执行管道的一部分，因此它们在不同的上下文中运行，并且没有内置的 `package.json` 脚本。

对于大多数开发者/团队来说，建议使用包脚本来构建和执行他们的 Nest 项目。你可以通过它们的选项（`--path`，`--webpack`，`--webpackPath`）和/或自定义 `tsc` 或 webpack 编译器选项文件（例如 `tsconfig.json`）来完全自定义这些脚本的行为。你也可以自由地运行一个完全自定义的构建过程来编译 TypeScript（甚至直接使用 `ts-node` 执行 TypeScript）。

#### 向后兼容性

由于 Nest 应用程序是纯 TypeScript 应用程序，以前的 Nest 构建/执行脚本将继续运行。你不需要升级它们。你可以选择在准备好时使用新的 `nest build` 和 `nest start` 命令，或者继续运行以前的或自定义的脚本。

#### 迁移

虽然你不需要做任何更改，但你可能希望迁移到使用新的 CLI 命令，而不是使用 `tsc-watch` 或 `ts-node` 等工具。在这种情况下，只需安装最新版本的 `@nestjs/cli`，既全局安装也本地安装：

```bash
$ npm install -g @nestjs/cli
$ cd  /some/project/root/folder
$ npm install -D @nestjs/cli
```

然后你可以将 `package.json` 中定义的 `scripts` 替换为以下内容：

```typescript
"build": "nest build",
"start": "nest start",
"start:dev": "nest start --watch",
"start:debug": "nest start --debug --watch",
```