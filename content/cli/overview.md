### 概览

[Nest CLI](https://github.com/nestjs/nest-cli) 是一个命令行界面工具，它帮助你初始化、开发和维护你的Nest应用程序。它以多种方式提供帮助，包括搭建项目框架、在开发模式下服务应用程序，以及构建和打包应用程序以供生产分发。它体现了最佳实践的架构模式，鼓励构建结构良好的应用程序。

#### 安装

**注意**：本指南描述使用 [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) 来安装包，包括Nest CLI。你也可以根据需要使用其他包管理器。使用npm时，你有几个选项可用于管理你的操作系统命令行解析 `nest` CLI二进制文件位置的方式。这里，我们描述使用 `-g` 选项全局安装 `nest` 二进制文件。这提供了一定程度的便利，也是我们在整个文档中假定的方法。注意，全局安装 **任何** `npm` 包意味着确保它们运行正确版本的责任在于用户。同时意味着如果你有不同项目，每个项目都将运行 **相同** 版本的CLI。一个合理的替代方案是使用 [npx](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md) 程序，内置于 `npm` cli（或类似功能与其他包管理器）以确保你运行 **受管理的版本** 的Nest CLI。我们建议你查阅 [npx文档](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md) 和/或你的DevOps支持人员以获取更多信息。

使用 `npm install -g` 命令全局安装CLI（有关全局安装的详细信息，请参见上面的 **注意**）。

```bash
$ npm install -g @nestjs/cli
```

> 信息提示 **提示** 另外，你可以使用此命令 `npx @nestjs/cli@latest` 无需全局安装CLI。

#### 基本工作流程

安装完成后，你可以直接从操作系统命令行通过 `nest` 可执行文件调用CLI命令。通过输入以下命令查看可用的 `nest` 命令：

```bash
$ nest --help
```

使用以下结构获取个别命令的帮助。将 `generate` 替换为 `new`、`add` 等任何命令，以获取该命令的详细帮助：

```bash
$ nest generate --help
```

要创建、构建并在开发模式下运行一个新的基本Nest项目，请转到应成为你新项目的父文件夹，并运行以下命令：

```bash
$ nest new my-nest-project
$ cd my-nest-project
$ npm run start:dev
```

在你的浏览器中打开 [http://localhost:3000](http://localhost:3000) 以查看新应用程序正在运行。当你更改任何源文件时，应用程序将自动重新编译并重新加载。

> 信息提示 **提示** 我们建议使用 [SWC构建器](/recipes/swc) 以获得更快的构建速度（比默认的TypeScript编译器性能高10倍）。

#### 项目结构

当你运行 `nest new` 时，Nest通过创建一个新文件夹并填充一组初始文件来生成样板应用程序结构。你可以继续在这个默认结构中工作，添加新组件，如本文档中所述。我们将通过 `nest new` 生成的项目结构称为 **标准模式**。Nest还支持用于管理多个项目和库的替代结构，称为 **单仓库模式**。

除了围绕 **构建** 过程的一些特定考虑（本质上，单仓库模式简化了有时可能出现的单仓库式项目结构的构建复杂性），以及内置的 [库](/cli/libraries) 支持外，其余的Nest功能和本文档同样适用于标准和单仓库模式项目结构。实际上，你可以随时从标准模式轻松切换到单仓库模式，所以当你还在了解Nest时，可以安全地推迟这个决定。

你可以使用任一模式来管理多个项目。以下是差异的快速总结：

| 特性                                               | 标准模式                                                      | 单仓库模式                                              |  
| ----------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------- |  
| 多个项目                                            | 分开的文件系统结构                                               | 单一文件系统结构                                        |
| `node_modules` & `package.json`                       | 分开的实例                                                    | 单仓库内共享                                       |
| 默认编译器                                        | `tsc`                                                              | webpack                                                    |  
| 编译器设置                                        | 分别指定                                                    | 单仓库默认设置，可以针对每个项目覆盖       |
| 配置文件如 `.eslintrc.js`, `.prettierrc` 等 | 分别指定                                                    | 单仓库内共享                                       |
| `nest build` 和 `nest start` 命令                | 目标默认自动设置为上下文中（唯一）的项目 | 目标默认为单仓库中的 **默认项目** |

阅读有关 [Workspaces](/cli/monorepo) 和 [Libraries](/cli/libraries) 的部分，以获取更多详细信息，帮助你决定哪种模式最适合你。

#### CLI命令语法

所有 `nest` 命令遵循相同的格式：

```bash
nest commandOrAlias requiredArg [optionalArg] [options]
```

例如：

```bash
$ nest new my-nest-project --dry-run
```

这里，`new` 是 _commandOrAlias_。`new` 命令有一个别名 `n`。`my-nest-project` 是 _requiredArg_。如果命令行上没有提供 _requiredArg_，`nest` 将提示输入。另外，`--dry-run` 有一个等效的简写形式 `-d`。考虑到这一点，以下命令与上述命令等效：

```bash
$ nest n my-nest-project -d
```

大多数命令和一些选项都有别名。尝试运行 `nest new --help` 查看这些选项和别名，并确认你对上述结构的理解。

#### 命令概览

运行 `nest <command> --help` 查看以下命令的命令特定选项。

有关每个命令的详细描述，请参见 [usage](/cli/usages)。

| 命令    | 别名 | 描述                                                                                    |
| ---------- | ----- | ---------------------------------------------------------------------------------------------- |
| `new`      | `n`   | 搭建一个新的 _标准模式_ 应用程序，包含运行所需的所有样板文件。          |
| `generate` | `g`   | 基于一个图生成和/或修改文件。                                          |
| `build`    |       | 将应用程序或工作区编译到输出文件夹。                                    |
| `start`    |       | 编译并运行应用程序（或工作区中的默认项目）。                          |
| `add`      |       | 导入一个已经打包为 **nest库** 的库，运行其安装图。 |
| `info`     | `i`   | 显示有关安装的nest包和其他有用系统信息。              |

#### 要求

Nest CLI需要一个内置 [国际化支持](https://nodejs.org/api/intl.html)（ICU）的Node.js二进制文件，例如从 [Node.js项目页面](https://nodejs.org/en/download) 获取的官方二进制文件。如果你遇到与ICU相关的错误，请检查你的二进制文件是否满足此要求。

```bash
node -p process.versions.icu
```

如果命令打印 `undefined`，则你的Node.js二进制文件没有国际化支持。