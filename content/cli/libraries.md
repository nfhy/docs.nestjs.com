### 库

许多应用程序需要解决相同的通用问题，或在不同上下文中重用模块化组件。Nest 提供了几种解决这个问题的方法，但每种方法都在不同的层次上工作，以帮助满足不同的架构和组织目标。

Nest [模块](/modules) 有助于提供执行上下文，使在单个应用程序中共享组件成为可能。模块还可以与 [npm](https://npmjs.com) 一起打包，创建可在不同项目中安装的可重用库。这可以是分发可配置、可重用的库的有效方式，这些库可以被不同的、松散连接或无关联的组织使用（例如，通过分发/安装第三方库）。

对于在紧密组织的团体内（例如，在公司/项目边界内）共享代码，拥有一种更轻量级的组件共享方法可能很有用。Monorepos 作为一种构造应运而生，使得这一点成为可能，而在 monorepo 中，**库** 提供了一种轻松、轻量级的代码共享方式。在 Nest monorepo 中，使用库可以轻松组装共享组件的应用程序。实际上，这鼓励将单体应用程序分解，并专注于构建和组合模块化组件的开发流程。

#### Nest 库

Nest 库是一个 Nest 项目，与应用程序不同，它不能独立运行。库必须导入到包含应用程序中，才能执行其代码。本节中描述的内置库支持仅适用于 **monorepos**（标准模式项目可以使用 npm 包实现类似的功能）。

例如，一个组织可能会开发一个 `AuthModule` 来管理身份验证，通过实现管理所有内部应用程序的公司政策。与其为每个应用程序单独构建该模块，或将代码物理打包为 npm 并要求每个项目安装它，不如将该模块定义为 monorepo 中的库。当以这种方式组织时，所有库模块的消费者都可以看到随着提交而更新的 `AuthModule` 的最新版本。这对于协调组件开发和组装，以及简化端到端测试具有重要意义。

#### 创建库

任何适合重用的功能都是作为库管理的候选对象。决定什么应该是库，什么应该是应用程序的一部分，是一个架构设计决策。创建库不仅仅是将代码从现有应用程序复制到新库。当作为库打包时，库代码必须与应用程序解耦。这可能需要**更多**的前期时间，并迫使你做出一些在更紧密耦合的代码中可能不会面临的设计决策。但当库可以用于加快多个应用程序的快速组装时，这些额外的努力就会得到回报。

要开始创建库，请运行以下命令：

```bash
$ nest g library my-library
```

当您运行命令时，`library` 模式会提示您为库输入一个前缀（别名）：

```bash
What prefix would you like to use for the library (default: @app)?
```

这会在您的工作区中创建一个名为 `my-library` 的新项目。
库类型项目，像应用程序类型项目一样，是使用模式生成到一个命名文件夹中的。库在 monorepo 根目录下的 `libs` 文件夹中管理。Nest 在第一次创建库时创建 `libs` 文件夹。

为库生成的文件与为应用程序生成的文件略有不同。以下是执行上述命令后 `libs` 文件夹的内容：

<div class="file-tree">
  <div class="item">libs</div>
  <div class="children">
    <div class="item">my-library</div>
    <div class="children">
      <div class="item">src</div>
      <div class="children">
        <div class="item">index.ts</div>
        <div class="item">my-library.module.ts</div>
        <div class="item">my-library.service.ts</div>
      </div>
      <div class="item">tsconfig.lib.json</div>
    </div>
  </div>
</div>

`nest-cli.json` 文件将在 `\"projects\"` 键下有一个新的库条目：

```javascript
...
{
    "my-library": {
      "type": "library",
      "root": "libs/my-library",
      "entryFile": "index",
      "sourceRoot": "libs/my-library/src",
      "compilerOptions": {
        "tsConfigPath": "libs/my-library/tsconfig.lib.json"
      }
    }
}
...
```

`nest-cli.json` 元数据中库和应用程序之间有两个不同之处：

- `\"type\"` 属性设置为 `\"library\"` 而不是 `\"application\"`。
- `\"entryFile\"` 属性设置为 `\"index\"` 而不是 `\"main\"`。

这些差异使构建过程能够适当地处理库。例如，库通过 `index.js` 文件导出其函数。

与应用程序类型项目一样，每个库都有自己的 `tsconfig.lib.json` 文件，该文件扩展了根（monorepo 范围）`tsconfig.json` 文件。如果需要，您可以修改此文件以提供库特定的编译器选项。

您可以使用 CLI 命令构建库：

```bash
$ nest build my-library
```

#### 使用库

有了自动生成的配置文件，使用库就变得简单直接了。我们如何将 `MyLibraryService` 从 `my-library` 库导入到 `my-project` 应用程序中呢？

首先，请注意，使用库模块与使用任何其他 Nest 模块相同。Monorepo 所做的是管理路径，使得导入库和生成构建变得透明。要使用 `MyLibraryService`，我们需要导入其声明模块。我们可以修改 `my-project/src/app.module.ts` 以导入 `MyLibraryModule`。

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MyLibraryModule } from '@app/my-library';

@Module({
  imports: [MyLibraryModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

请注意，我们在 ES 模块 `import` 行中使用了 `@app` 的路径别名，这是我们之前使用 `nest g library` 命令提供的 `prefix`。在底层，Nest 通过 tsconfig 路径映射处理这个问题。当添加库时，Nest 更新全局（monorepo）`tsconfig.json` 文件的 `\"paths\"` 键如下：

```javascript
"paths": {
    "@app/my-library": [
        "libs/my-library/src"
    ],
    "@app/my-library/*": [
        "libs/my-library/src/*"
    ]
}
```

简而言之，monorepo 和库功能的结合使得将库模块包含在应用程序中变得简单直观。

这种机制同样使得构建和部署组合库的应用程序成为可能。一旦您导入了 `MyLibraryModule`，运行 `nest build` 会自动处理所有模块解析，并将应用程序与任何库依赖项一起捆绑，以进行部署。monorepo 的默认编译器是 **webpack**，因此生成的分发文件是一个将所有转译后的 JavaScript 文件捆绑到一个文件中的单个文件。您也可以按照 <a href="https://docs.nestjs.com/cli/monorepo#global-compiler-options">此处</a> 描述切换到 `tsc`。