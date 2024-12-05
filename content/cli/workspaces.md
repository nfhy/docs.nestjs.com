### 工作区

Nest 提供了两种代码组织模式：

- **标准模式**：适用于构建单个项目为中心的应用程序，这些应用程序拥有自己的依赖项和设置，不需要优化模块共享或复杂构建。这是默认模式。
- **单仓库模式**：此模式将代码工件视为轻量级**单仓库**的一部分，可能更适合开发团队和/或多项目环境。它自动化了构建过程的部分，使得创建和组合模块化组件变得容易，促进了代码重用，使集成测试更容易，便于共享项目范围的工件，如 `eslint` 规则和其他配置策略，并且比使用 github 子模块等替代方案更容易使用。单仓库模式采用**工作区**的概念，在 `nest-cli.json` 文件中协调单仓库组件之间的关系。

需要注意的是，Nest 的几乎所有特性都与您的代码组织模式无关。这个选择的**唯一**影响是您的项目是如何组合以及如何生成构建工件的。所有其他功能，从 CLI 到核心模块再到附加模块，在两种模式下的工作方式相同。

此外，您可以随时从**标准模式**切换到**单仓库模式**，因此您可以推迟这个决定，直到一种或另一种方法的好处变得更加清晰。

#### 标准模式

当您运行 `nest new` 时，Nest 会为您创建一个新的**项目**，使用内置的架构。Nest 执行以下操作：

1. 创建一个新文件夹，对应于您提供给 `nest new` 的 `name` 参数。
2. 用对应于最小基本 Nest 应用程序的默认文件填充该文件夹。您可以在 [typescript-starter](https://github.com/nestjs/typescript-starter) 仓库中查看这些文件。
3. 提供额外的文件，如 `nest-cli.json`、`package.json` 和 `tsconfig.json`，配置并启用编译、测试和提供应用程序的各种工具。

从那里开始，您可以修改起始文件，添加新组件，添加依赖项（例如，`npm install`），并按照本文档的其余部分所涵盖的方式开发您的应用程序。

#### 单仓库模式

要启用单仓库模式，您从**标准模式**结构开始，并添加**项目**。项目可以是完整的**应用程序**（您可以通过命令 `nest generate app` 将其添加到工作区）或**库**（您可以通过命令 `nest generate library` 将其添加到工作区）。我们将在下面讨论这些特定类型的项目组件的详细信息。现在需要注意的关键点是，将**项目**添加到现有标准模式结构的行为将**将其转换**为单仓库模式。让我们看一个例子。

如果我们运行：

```bash
$ nest new my-project
```

我们构建了一个**标准模式**结构，文件夹结构如下所示：

<div class="file-tree">
  <div class="item">node_modules</div>
  <div class="item">src</div>
  <div class="children">
    <div class="item">app.controller.ts</div>
    <div class="item">app.module.ts</div>
    <div class="item">app.service.ts</div>
    <div class="item">main.ts</div>
  </div>
  <div class="item">nest-cli.json</div>
  <div class="item">package.json</div>
  <div class="item">tsconfig.json</div>
  <div class="item">.eslintrc.js</div>
</div>

我们可以按照以下方式将此转换为单仓库模式结构：

```bash
$ cd my-project
$ nest generate app my-app
```

此时，`nest` 将现有结构转换为**单仓库模式**结构。这导致一些重要的变化。文件夹结构现在看起来像这样：

<div class="file-tree">
  <div class="item">apps</div>
  <div class="children">
    <div class="item">my-app</div>
    <div class="children">
      <div class="item">src</div>
      <div class="children">
        <div class="item">app.controller.ts</div>
        <div class="item">app.module.ts</div>
        <div class="item">app.service.ts</div>
        <div class="item">main.ts</div>
      </div>
      <div class="item">tsconfig.app.json</div>
    </div>
    <div class="item">my-project</div>
    <div class="children">
      <div class="item">src</div>
      <div class="children">
        <div class="item">app.controller.ts</div>
        <div class="item">app.module.ts</div>
        <div class="item">app.service.ts</div>
        <div class="item">main.ts</div>
      </div>
      <div class="item">tsconfig.app.json</div>
    </div>
  </div>
  <div class="item">nest-cli.json</div>
  <div class="item">package.json</div>
  <div class="item">tsconfig.json</div>
  <div class="item">.eslintrc.js</div>
</div>

`generate app` 架构重新组织了代码 - 将每个**应用程序**项目移动到 `apps` 文件夹下，并在每个项目的根文件夹中添加了项目特定的 `tsconfig.app.json` 文件。我们的原始 `my-project` 应用程序已成为单仓库的**默认项目**，现在与刚刚添加的 `my-app` 同级，位于 `apps` 文件夹下。我们将在下面讨论默认项目。

> 错误 **警告** 将标准模式结构转换为单仓库仅适用于遵循规范 Nest 项目结构的项目。具体来说，在转换过程中，架构尝试将 `src` 和 `test` 文件夹重新定位到项目文件夹下的 `apps` 文件夹中。如果项目不使用此结构，转换将失败或产生不可靠的结果。

#### 工作区项目

单仓库使用工作区的概念来管理其成员实体。工作区由**项目**组成。项目可以是：

- 一个**应用程序**：包括用于启动应用程序的 `main.ts` 文件的完整 Nest 应用程序。除了编译和构建考虑因素外，工作区内的应用程序类型项目在功能上与标准模式结构中的应用程序相同。
- 一个**库**：库是一种打包通用功能集（模块、提供者、控制器等）的方式，这些功能需要被组合到其他项目中才能运行。库不能单独运行，并且没有 `main.ts` 文件。更多关于库的信息[在这里](/cli/libraries)。

所有工作区都有一个**默认项目**（应该是应用程序类型的项目）。这由顶级 `\"root\"` 属性在 `nest-cli.json` 文件中定义，指向默认项目的根（有关更多详细信息，请参见下面的 [CLI 属性](/cli/monorepo#cli-properties)）。通常，这是您开始的**标准模式**应用程序，后来使用 `nest generate app` 转换为单仓库。当您按照这些步骤操作时，此属性会自动填充。

默认项目由 `nest` 命令如 `nest build` 和 `nest start` 在没有提供项目名称时使用。

例如，在上述单仓库结构中，运行

```bash
$ nest start
```

将启动 `my-project` 应用程序。要启动 `my-app`，我们将使用：

```bash
$ nest start my-app
```

#### 应用程序

应用程序类型的项目，或我们可能非正式地称为只是“应用程序”，是可以运行和部署的完整 Nest 应用程序。您使用 `nest generate app` 生成应用程序类型的项目。

此命令自动生成项目骨架，包括来自 [typescript starter](https://github.com/nestjs/typescript-starter) 的标准 `src` 和 `test` 文件夹。与标准模式不同，单仓库中的应用程序项目没有任何包依赖项（`package.json`）或其他项目配置文件，如 `.prettierrc` 和 `.eslintrc.js`。相反，使用单仓库范围的依赖项和配置文件。

然而，架构确实在项目的根文件夹中生成了一个项目特定的 `tsconfig.app.json` 文件。此配置文件自动设置适当的构建选项，包括正确设置编译输出文件夹。该文件扩展了顶级（单仓库）`tsconfig.json` 文件，因此您可以单仓库范围管理全局设置，但如果需要，可以在项目级别覆盖它们。

#### 库

如上所述，库类型的项目，或简单地称为“库”，是需要被组合到应用程序中才能运行的 Nest 组件包。您使用 `nest generate library` 生成库类型的项目。决定什么属于库是一个架构设计决策。我们在[库](/cli/libraries)章节中详细讨论库。

#### CLI 属性

Nest 将组织、构建和部署标准和单仓库结构项目所需的元数据保存在 `nest-cli.json` 文件中。Nest 在您添加项目时自动添加和更新此文件，因此您通常不需要考虑或编辑其内容。然而，有一些设置您可能想要手动更改，因此了解文件的概览是有帮助的。

在运行上述步骤创建单仓库后，我们的 `nest-cli.json` 文件如下所示：

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "apps/my-project/src",
  "monorepo": true,
  "root": "apps/my-project",
  "compilerOptions": {
    "webpack": true,
    "tsConfigPath": "apps/my-project/tsconfig.app.json"
  },
  "projects": {
    "my-project": {
      "type": "application",
      "root": "apps/my-project",
      "entryFile": "main",
      "sourceRoot": "apps/my-project/src",
      "compilerOptions": {
        "tsConfigPath": "apps/my-project/tsconfig.app.json"
      }
    },
    "my-app": {
      "type": "application",
      "root": "apps/my-app",
      "entryFile": "main",
      "sourceRoot": "apps/my-app/src",
      "compilerOptions": {
        "tsConfigPath": "apps/my-app/tsconfig.app.json"
      }
    }
  }
}
```

文件分为几个部分：

- 一个全局部分，顶级属性控制标准和单仓库范围的设置。
- 一个顶级属性（`"projects"`），包含有关每个项目的元数据。此部分仅存在于单仓库模式结构中。

顶级属性如下：

- `"collection"`：指向用于生成组件的架构集合；您通常不应该更改此值。
- `"sourceRoot"`：指向标准模式结构中单个项目的源代码根，或单仓库模式结构中的**默认项目**。
- `"compilerOptions"`：一个映射，键指定编译器选项，值指定选项设置；详见下文。
- `"generateOptions"`：一个映射，键指定全局生成选项，值指定选项设置；详见下文。
- `"monorepo"`：（单仓库仅）对于单仓库模式结构，此值始终为 `true`。
- `"root"`：（单仓库仅）指向**默认项目**的项目根。

#### 全局编译器选项

这些属性指定要使用的编译器以及影响**任何**编译步骤的各种选项，无论是作为 `nest build` 或 `nest start` 的一部分，以及无论编译器是 `tsc` 还是 webpack。

| 属性名称       | 属性值类型 | 描述                                                                                                                                                                                                                                                               |
| -------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `webpack`      | boolean    | 如果为 `true`，则使用 [webpack 编译器](https://webpack.js.org/)。如果为 `false` 或不存在，则使用 `tsc`。在单仓库模式中，默认为 `true`（使用 webpack），在标准模式中，默认为 `false`（使用 `tsc`）。详见下文。（已弃用：请改用 `builder`） |
| `tsConfigPath` | string     | （**单仓库仅**）指向包含将在调用 `nest build` 或 `nest start` 时使用的 `tsconfig.json` 设置的文件（例如，当构建或启动默认项目时）。                                                                                                                |
| `webpackConfigPath` | string | 指向 webpack 选项文件。如果未指定，Nest 将寻找文件 `webpack.config.js`。详见下文以获取更多详细信息。                                                                                                                                                    |
| `deleteOutDir` | boolean    | 如果为 `true`，则每当编译器被调用时，它将首先删除编译输出目录（如 `tsconfig.json` 中配置的，默认为 `./dist`）。                                                                                                                               |
| `assets`       | array      | 启用在编译步骤开始时自动分发非 TypeScript 资产（在 `--watch` 模式下的增量编译中不发生资产分发）。详见下文。                                                                                                                                                    |
| `watchAssets`  | boolean    | 如果为 `true`，则运行在 watch 模式下，监视**所有**非 TypeScript 资产。（有关更细粒度的资产监视控制，请参见[资产](cli/monorepo#assets)部分）。                                                                                                            |
| `manualRestart`| boolean    | 如果为 `true`，则启用快捷方式 `rs` 手动重启服务器。默认值为 `false`。                                                                                                                                                                                                |
| `builder`      | string/object | 指示 CLI 用于编译项目的 `builder`（`tsc`、`swc` 或 `webpack`）。要自定义构建器的行为，您可以传递一个包含两个属性的对象：`type`（`tsc`、`swc` 或 `webpack`）和 `options`。                                                              |
| `typeCheck`    | boolean    | 如果为 `true`，则启用 SWC 驱动项目（当 `builder` 为 `swc` 时）的类型检查。默认值为 `false`。                                                                                                                                                              |

#### 全局生成选项

这些属性指定 `nest generate` 命令使用的默认生成选项。

| 属性名称 | 属性值类型 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| -------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `spec`   | boolean _or_ object | 如果值是布尔值，`true` 值默认启用 `spec` 生成，`false` 值默认禁用它。CLI 命令行上传递的标志会覆盖此设置，项目特定的 `generateOptions` 设置（更多信息见下文）也是如此。如果值是对象，每个键代表一个架构名称，布尔值确定是否默认启用该特定架构的 spec 生成。 |
| `flat`   | boolean    | 如果为 true，则所有生成命令将生成一个平面结构                                                                                                                                                                                                                                                                                                                                                                                    |

以下示例使用布尔值指定默认情况下应为所有项目禁用 spec 文件生成：

```javascript
{
  "generateOptions": {
    "spec": false
  },
  ...
}
```

以下示例使用布尔值指定平面文件生成应为所有项目默认：

```javascript
{
  "generateOptions": {
    "flat": true
  },
  ...
}
```

在以下示例中，仅禁用 `service` 架构的 spec 文件生成（例如，`nest generate service...`）：

```javascript
{
  "generateOptions": {
    "spec": {
      "service": false
    }
  },
  ...
}
```

> 警告 **警告** 当指定 `spec` 为对象时，生成架构的键当前不支持自动别名处理。这意味着指定一个键为 `service: false` 并尝试通过别名 `s` 生成一个服务，spec 仍然会被生成。为了确保正常和别名都能按预期工作，请指定正常命令名称以及别名，如下所示。
>

```javascript
{
  "generateOptions": {
    "spec": {
      "service": false,
      "s": false
    }
  },
  ...
}
```

#### 项目特定的生成选项

除了提供全局生成选项外，您还可以指定项目特定的生成选项。项目特定的生成选项遵循与全局生成选项完全相同的格式，但直接在每个项目上指定。

项目特定的生成选项覆盖全局生成选项。

```javascript
{
  "projects": {
    "cats-project": {
      "generateOptions": {
        "spec": {
          "service": false
        }
      },
      ...
    }
  },
  ...
}
```

> 警告 **警告** 生成选项的优先级顺序如下：CLI 命令行上指定的选项优先于项目特定选项。项目特定选项覆盖全局选项。

#### 指定编译器

不同默认编译器的原因是，对于较大的项目（例如，在单仓库中更典型），webpack 可以在构建时间和生成一个包含所有项目组件的单个文件捆绑包方面具有显著优势。如果您希望生成单独的文件，请将 `"webpack"` 设置为 `false`，这将导致构建过程使用 `tsc`（或 `swc`）。

#### Webpack 选项

Webpack 选项文件可以包含标准的 [webpack 配置选项](https://webpack.js.org/configuration/)。例如，要告诉 webpack 捆绑 `node_modules`（默认情况下是排除的），请在 `webpack.config.js` 中添加以下内容：

```javascript
module.exports = {
  externals: [],
};
```

由于 webpack 配置文件是一个 JavaScript 文件，您甚至可以暴露一个函数，该函数接受默认选项并返回一个修改后的对象：

```javascript
module.exports = function (options) {
  return {
    ...options,
    externals: [],
  };
};
```

#### 资产

TypeScript 编译自动将编译输出（`.js` 和 `.d.ts` 文件）分发到指定的输出目录。分发非 TypeScript 文件，如 `.graphql` 文件、`images`、`.html` 文件和其他资产，也可能很方便。这允许您将 `nest build`（和任何初始编译步骤）视为轻量级**开发构建**步骤，在此过程中，您可能正在编辑非 TypeScript 文件，并迭代编译和测试。

资产应位于 `src` 文件夹中，否则它们将不会被复制。

`assets` 键的值应该是一个数组，指定要分发的文件。元素可以是简单的字符串，带有 `glob` 样式的文件规格，例如：

```typescript
"assets": ["**/*.graphql"],
"watchAssets": true,
```

对于更精细的控制，元素可以是具有以下键的对象：

- `"include"`：要分发的资产的 `glob` 样式文件规格
- `"exclude"`：要从 `include` 列表中**排除**的资产的 `glob` 样式文件规格
- `"outDir"`：一个字符串，指定资产应该分发到哪里的路径（相对于根文件夹）。默认为编译器输出配置的相同输出目录。
- `"watchAssets"`：布尔值；如果为 `true`，则运行在 watch 模式下监视指定的资产

例如：

```typescript
"assets": [
  { "include": "**/*.graphql", "exclude": "**/omitted.graphql", "watchAssets": true },
]
```

> 警告 **警告** 在顶级 `compilerOptions` 属性中设置 `watchAssets` 会覆盖 `assets` 属性中的任何 `watchAssets` 设置。

#### 项目属性

此元素仅存在于单仓库模式结构中。您通常不应该编辑这些属性，因为它们被 Nest 用来在单仓库中定位项目及其配置选项。