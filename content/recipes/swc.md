### SWC

[SWC](https://swc.rs/)（Speedy Web Compiler）是一个可扩展的基于Rust的平台，可以用于编译和打包。

使用SWC与Nest CLI相结合是加速开发过程的一个简单而有效的方法。

> 信息 **提示** SWC的速度大约是默认TypeScript编译器的**20倍**。

#### 安装

首先，安装几个包：

```bash
$ npm i --save-dev @swc/cli @swc/core
```

#### 开始使用

安装过程完成后，您可以使用`swc`构建器与Nest CLI一起使用，如下所示：

```bash
$ nest start -b swc
# 或者 nest start --builder swc
```

> 信息 **提示** 如果您的仓库是单体仓库，请查看[这一节](/recipes/swc#monorepo)。

您也可以不传递`-b`标志，而是直接在`nest-cli.json`文件中将`compilerOptions.builder`属性设置为`"swc"`，如下所示：

```json
{
  "compilerOptions": {
    "builder": "swc"
  }
}
```

要自定义构建器的行为，您可以传递一个包含两个属性的对象，`type`（`"swc"`）和`options`，如下所示：

```json
"compilerOptions": {
  "builder": {
    "type": "swc",
    "options": {
      "swcrcPath": "infrastructure/.swcrc",
    }
  }
}
```

要运行应用程序的监视模式，请使用以下命令：

```bash
$ nest start -b swc -w
# 或者 nest start --builder swc --watch
```

#### 类型检查

SWC本身不执行任何类型检查（与默认的TypeScript编译器不同），因此要启用它，您需要使用`--type-check`标志：

```bash
$ nest start -b swc --type-check
```

此命令将指示Nest CLI与SWC一起异步运行`tsc`的`noEmit`模式，执行类型检查。同样，您也可以不传递`--type-check`标志，而是直接在`nest-cli.json`文件中将`compilerOptions.typeCheck`属性设置为`true`，如下所示：

```json
{
  "compilerOptions": {
    "builder": "swc",
    "typeCheck": true
  }
}
```

#### CLI插件（SWC）

`--type-check`标志将自动执行**NestJS CLI插件**并生成一个序列化的元数据文件，然后可以在运行时由应用程序加载。

#### SWC配置

SWC构建器预配置以满足NestJS应用程序的要求。但是，您可以通过在根目录创建`.swcrc`文件并根据您的需要调整选项来自定义配置。

```json
{
  "$schema": "https://json.schemastore.org/swcrc",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

#### 单体仓库

如果您的仓库是单体仓库，那么您不能使用`swc`构建器，而是需要配置`webpack`以使用`swc-loader`。

首先，安装所需的包：

```bash
$ npm i --save-dev swc-loader
```

安装完成后，在应用程序的根目录创建一个`webpack.config.js`文件，内容如下：

```js
const swcDefaultConfig = require('@nestjs/cli/lib/compiler/defaults/swc-defaults').swcDefaultsFactory().swcOptions;

module.exports = {
  module: {
    rules: [
      {
        test: /\.ts$/,
        exclude: /node_modules/,
        use: {
          loader: 'swc-loader',
          options: swcDefaultConfig,
        },
      },
    ],
  },
};
```

#### 单体仓库和CLI插件

现在如果您使用CLI插件，`swc-loader`不会自动加载它们。相反，您必须创建一个单独的文件手动加载它们。

为此，声明一个`generate-metadata.ts`文件在`main.ts`文件附近，内容如下：

```ts
import { PluginMetadataGenerator } from '@nestjs/cli/lib/compiler/plugins/plugin-metadata-generator';
import { ReadonlyVisitor } from '@nestjs/swagger/dist/plugin';

const generator = new PluginMetadataGenerator();
generator.generate({
  visitors: [new ReadonlyVisitor({ introspectComments: true, pathToSource: __dirname })],
  outputDir: __dirname,
  watch: true,
  tsconfigPath: 'apps/<name>/tsconfig.app.json',
});
```

> 信息 **提示** 在这个例子中我们使用了`@nestjs/swagger`插件，但您可以使用任何您选择的插件。

`generate()`方法接受以下选项：

|                    |                                                                                                |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| `watch`            | 是否监视项目以侦测变化。                                                      |
| `tsconfigPath`     | `tsconfig.json`文件的路径。相对于当前工作目录（`process.cwd()`）。 |
| `outputDir`        | 保存元数据文件的目录路径。                                   |
| `visitors`         | 用于生成元数据的访问者数组。                                   |
| `filename`         | 元数据文件的名称。默认为`metadata.ts`。                                      |
| `printDiagnostics` | 是否在控制台打印诊断信息。默认为`true`。                                |

最后，您可以在单独的终端窗口中使用以下命令运行`generate-metadata`脚本：

```bash
$ npx ts-node src/generate-metadata.ts
# 或者 npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

#### 常见陷阱

如果您在应用程序中使用TypeORM/MikroORM或任何其他ORM，您可能会遇到循环导入问题。SWC不擅长处理**循环导入**，因此您应该使用以下变通方法：

```typescript
@Entity()
export class User {
  @OneToOne(() => Profile, (profile) => profile.user)
  profile: Relation<Profile>; // <--- 这里使用“Relation<>”类型而不是仅仅是“Profile”
}
```

> 信息 **提示** `Relation`类型是从`typeorm`包导出的。

这样做可以防止属性的类型被保存在转译后的代码中的属性元数据中，从而避免循环依赖问题。

如果您的ORM没有提供类似的变通方法，您可以自己定义包装器类型：

```typescript
/**
 * 用于规避ESM模块循环依赖问题
 * 由反射元数据保存属性类型引起的。
 */
export type WrapperType<T> = T; // WrapperType === Relation
```

对于您项目中的所有[循环依赖注入](/fundamentals/circular-dependency)，您也需要使用上述自定义包装器类型：

```typescript
@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => ProfileService))
    private readonly profileService: WrapperType<ProfileService>,
  ) {};
}
```

### Jest + SWC

要将SWC与Jest一起使用，您需要安装以下包：

```bash
$ npm i --save-dev jest @swc/core @swc/jest
```

安装完成后，更新`package.json`/`jest.config.js`文件（取决于您的配置），内容如下：

```json
{
  "jest": {
    "transform": {
      "^.+\\.(t|j)s?$": ["@swc/jest"]
    }
  }
}
```

此外，您还需要在`.swcrc`文件中添加`legacyDecorator`和`decoratorMetadata`这两个`transform`属性：

```json
{
  "$schema": "https://json.schemastore.org/swcrc",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "transform": {
      "legacyDecorator": true,
      "decoratorMetadata": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

如果您的项目中使用了NestJS CLI插件，您将不得不手动运行`PluginMetadataGenerator`。请导航至[这一节](/recipes/swc#monorepo-and-cli-plugins)了解更多信息。

### Vitest

[Vitest](https://vitest.dev/)是一个快速且轻量级的测试运行器，旨在与Vite一起工作。它提供了一个现代、快速且易于使用的测试解决方案，可以与NestJS项目集成。

#### 安装

首先，安装所需的包：

```bash
$ npm i --save-dev vitest unplugin-swc @swc/core @vitest/coverage-v8
```

#### 配置

在应用程序的根目录创建一个`vitest.config.ts`文件，内容如下：

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    root: './',
  },
  plugins: [
    // 这是构建测试文件所必需的SWC插件
    swc.vite({
      // 显式设置模块类型以避免从`.swcrc`配置文件继承此值
      module: { type: 'es6' },
    }),
  ],
});
```

此配置文件设置了Vitest环境、根目录和SWC插件。您还应该为端到端测试创建一个单独的配置文件，并添加一个`include`字段，指定测试路径的正则表达式：

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    root: './',
  },
  plugins: [swc.vite()],
});
```

此外，您可以设置`alias`选项以支持测试中的TypeScript路径：

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    alias: {
      '@src': './src',
      '@test': './test',
    },
    root: './',
  },
  resolve: {
    alias: {
      '@src': './src',
      '@test': './test',
    },
  },
  plugins: [swc.vite()],
});
```

#### 更新E2E测试中的导入

将任何使用`import * as request from 'supertest'`的E2E测试导入更改为`import request from 'supertest'`。这是必要的，因为Vitest在与Vite打包时，期望supertest有一个默认导入。使用命名空间导入可能会在这个特定设置中引起问题。

最后，更新您的package.json文件中的测试脚本，如下所示：

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:cov": "vitest run --coverage",
    "test:debug": "vitest --inspect-brk --inspect --logHeapUsage --threads=false",
    "test:e2e": "vitest run --config ./vitest.config.e2e.ts"
  }
}
```

这些脚本配置Vitest以运行测试、监视更改、生成代码覆盖率报告和调试。test:e2e脚本专门用于使用自定义配置文件运行E2E测试。

有了这个设置，您现在可以享受在NestJS项目中使用Vitest的好处，包括更快的测试执行和更现代的测试体验。

> 信息 **提示** 您可以在这个[仓库](https://github.com/TrilonIO/nest-vitest)中查看一个工作示例。