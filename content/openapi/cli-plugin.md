### CLI 插件

[TypeScript](https://www.typescriptlang.org/docs/handbook/decorators.html)的元数据反射系统有一些限制，这使得它无法确定一个类包含哪些属性，或者识别给定属性是可选的还是必需的。然而，这些限制中的一些可以在编译时解决。Nest 提供了一个插件，增强了 TypeScript 编译过程，以减少所需的样板代码数量。

> 信息 **提示** 这个插件是 **可选的**。如果你愿意，你可以手动声明所有装饰器，或者只在需要的地方声明特定的装饰器。

#### 概览

Swagger 插件将自动执行以下操作：

- 除非使用了 `@ApiHideProperty`，否则为所有 DTO 属性标注 `@ApiProperty`
- 根据问号设置 `required` 属性（例如 `name?: string` 将设置 `required: false`）
- 根据类型设置 `type` 或 `enum` 属性（支持数组）
- 根据分配的默认值设置 `default` 属性
- 根据 `class-validator` 装饰器设置多个验证规则（如果 `classValidatorShim` 设置为 `true`）
- 为每个端点添加响应装饰器，带有适当的状态和 `type`（响应模型）
- 根据注释生成属性和端点的描述（如果 `introspectComments` 设置为 `true`）
- 根据注释生成属性的示例值（如果 `introspectComments` 设置为 `true`）

请注意，你的文件名**必须**有以下后缀之一：`['.dto.ts', '.entity.ts']`（例如，`create-user.dto.ts`），以便插件进行分析。

如果你使用不同的后缀，你可以通过指定 `dtoFileNameSuffix` 选项来调整插件的行为（见下文）。

以前，如果你想提供与 Swagger UI 交互的体验，你必须复制很多代码，以让包知道你的模型/组件应该如何在规范中声明。例如，你可以定义一个简单的 `CreateUserDto` 类，如下所示：

```typescript
export class CreateUserDto {
  @ApiProperty()
  email: string;

  @ApiProperty()
  password: string;

  @ApiProperty({ enum: RoleEnum, default: [], isArray: true })
  roles: RoleEnum[] = [];

  @ApiProperty({ required: false, default: true })
  isEnabled?: boolean = true;
}
```

虽然对于中等规模的项目来说，这并不是一个显著的问题，但一旦你有了大量的类，它就会变得冗长且难以维护。

通过[启用 Swagger 插件](/openapi/cli-plugin#using-the-cli-plugin)，上述类定义可以简单地声明为：

```typescript
export class CreateUserDto {
  email: string;
  password: string;
  roles: RoleEnum[] = [];
  isEnabled?: boolean = true;
}
```

> 信息 **注意** Swagger 插件将从 TypeScript 类型和 class-validator 装饰器中派生出 `@ApiProperty()` 注解。这有助于为生成的 Swagger UI 文档清晰地描述你的 API。然而，运行时的验证仍然由 class-validator 装饰器处理。因此，继续使用 `IsEmail()`、`IsNumber()` 等验证器是必需的。

因此，如果你想依赖自动注释来生成文档，并且仍然希望进行运行时验证，那么 class-validator 装饰器仍然是必要的。

> 信息 **提示** 当在 DTO 中使用 [映射类型工具](https://docs.nestjs.com/openapi/mapped-types)（如 `PartialType`）时，从 `@nestjs/swagger` 而不是 `@nestjs/mapped-types` 导入它们，以便插件能够识别模式。

插件会根据 **抽象语法树** 动态添加适当的装饰器。因此，你不必在代码中到处使用 `@ApiProperty` 装饰器。

> 信息 **提示** 插件会自动生成任何缺失的 Swagger 属性，但如果你需要覆盖它们，你只需通过 `@ApiProperty()` 显式设置它们。

#### 注释内省

启用注释内省功能后，CLI 插件将根据注释生成属性的描述和示例值。

例如，给定一个 `roles` 属性的例子：

```typescript
/**
 * 用户的角色列表
 * @example ['admin']
 */
@ApiProperty({
  description: `用户的角色列表`,
  example: ['admin'],
})
roles: RoleEnum[] = [];
```

你必须同时复制描述和示例值。启用 `introspectComments` 后，CLI 插件可以提取这些注释，并自动为属性提供描述（如果定义了示例，则包括示例）。现在，上述属性可以简单地声明如下：

```typescript
/**
 * 用户的角色列表
 * @example ['admin']
 */
roles: RoleEnum[] = [];
```

有 `dtoKeyOfComment` 和 `controllerKeyOfComment` 插件选项可用于自定义插件如何将值分配给 `ApiProperty` 和 `ApiOperation` 装饰器。下面是一个例子：

```typescript
export class SomeController {
  /**
   * 创建某个资源
   */
  @Post()
  create() {}
}
```

这相当于以下指令：

```typescript
@ApiOperation({ summary: "创建某个资源" })
```

> 信息 **提示** 对于模型，同样的逻辑适用，但用于 `ApiProperty` 装饰器。

对于控制器，你不仅可以提供摘要，还可以提供描述（备注）、标签（如 `@deprecated`）和响应示例，如下所示：

```ts
/**
 * 创建一只新猫
 *
 * @remarks 此操作允许你创建一只新猫。
 *
 * @deprecated
 * @throws {500} 出了点问题。
 * @throws {400} 错误的请求。
 */
@Post()
async create(): Promise<Cat> {}
```

#### 使用 CLI 插件

要启用插件，请打开 `nest-cli.json`（如果你使用 [Nest CLI](/cli/overview)）并添加以下 `plugins` 配置：

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/swagger"]
  }
}
```

你可以使用 `options` 属性来自定义插件的行为。

```javascript
"plugins": [
  {
    "name": "@nestjs/swagger",
    "options": {
      "classValidatorShim": false,
      "introspectComments": true
    }
  }
]
```

`options` 属性必须满足以下接口：

```typescript
export interface PluginOptions {
  dtoFileNameSuffix?: string[];
  controllerFileNameSuffix?: string[];
  classValidatorShim?: boolean;
  dtoKeyOfComment?: string;
  controllerKeyOfComment?: string;
  introspectComments?: boolean;
}
```

<table>
  <tr>
    <th>选项</th>
    <th>默认</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>dtoFileNameSuffix</code></td>
    <td><code>['.dto.ts', '.entity.ts']</code></td>
    <td>DTO（数据传输对象）文件后缀</td>
  </tr>
  <tr>
    <td><code>controllerFileNameSuffix</code></td>
    <td><code>.controller.ts</code></td>
    <td>控制器文件后缀</td>
  </tr>
  <tr>
    <td><code>classValidatorShim</code></td>
    <td><code>true</code></td>
    <td>如果设置为 true，则模块将重用 <code>class-validator</code> 验证装饰器（例如 <code>@Max(10)</code> 将添加 <code>max: 10</code> 到模式定义）</td>
  </tr>
  <tr>
    <td><code>dtoKeyOfComment</code></td>
    <td><code>'description'</code></td>
    <td>在 <code>ApiProperty</code> 上设置注释文本的属性键。</td>
  </tr>
  <tr>
    <td><code>controllerKeyOfComment</code></td>
    <td><code>'summary'</code></td>
    <td>在 <code>ApiOperation</code> 上设置注释文本的属性键。</td>
  </tr>
  <tr>
    <td><code>introspectComments</code></td>
    <td><code>false</code></td>
    <td>如果设置为 true，插件将根据注释生成属性的描述和示例值</td>
  </tr>
</table>

更新插件选项后，请确保删除 `/dist` 文件夹并重新构建你的应用程序。

如果你不使用 CLI，而是有一个自定义的 `webpack` 配置，你可以将这个插件与 `ts-loader` 结合使用：

```javascript
getCustomTransformers: (program: any) => ({
  before: [require('@nestjs/swagger/plugin').before({}, program)]
}),
```

#### SWC 构建器

对于标准设置（非单仓库），要使用 CLI 插件与 SWC 构建器一起使用，你需要启用类型检查，如[此处](/recipes/swc#type-checking)所述。

```bash
$ nest start -b swc --type-check
```

对于单仓库设置，请按照[此处](/recipes/swc#monorepo-and-cli-plugins)的说明操作。

```bash
$ npx ts-node src/generate-metadata.ts
# 或 npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

现在，序列化的元数据文件必须由 `SwaggerModule#loadPluginMetadata` 方法加载，如下所示：

```typescript
import metadata from './metadata'; // <-- 文件由 "PluginMetadataGenerator" 自动生成

await SwaggerModule.loadPluginMetadata(metadata); // <-- 这里
const document = SwaggerModule.createDocument(app, config);
```

#### 与 `ts-jest`（端到端测试）集成

要运行端到端测试，`ts-jest` 会在内存中即时编译你的源代码文件。这意味着它不使用 Nest CLI 编译器，不应用任何插件，也不执行 AST 转换。

要启用插件，请在端到端测试目录中创建以下文件：

```javascript
const transformer = require('@nestjs/swagger/plugin');

module.exports.name = 'nestjs-swagger-transformer';
// 每当你更改下面的配置时，都应该更改版本号 - 否则，jest 将不会检测到更改
module.exports.version = 1;

module.exports.factory = (cs) => {
  return transformer.before(
    {
      // @nestjs/swagger/plugin 选项（可以为空）
    },
    cs.program, // 对于旧版本的 Jest（<= v27），使用 "cs.tsCompiler.program"
  );
};
```

有了这个，在你的 `jest` 配置文件中导入 AST 转换器。默认情况下（在起始应用程序中），端到端测试配置文件位于 `test` 文件夹中，名为 `jest-e2e.json`。

```json
{
  ... // 其他配置
  "globals": {
    "ts-jest": {
      "astTransformers": {
        "before": ["<path to the file created above>"]
      }
    }
  }
}
```

如果你使用 `jest@^29`，那么使用下面的代码片段，因为之前的方法是被弃用的。

```json
{
  ... // 其他配置
  "transform": {
    "^.+\\\\.(t|j)s$": [
      "ts-jest",
      {
        "astTransformers": {
          "before": ["<path to the file created above>"]
        }
      }
    ]
  }
}
```

#### 排除 `jest`（端到端测试）故障

如果 `jest` 似乎没有拾取你的配置更改，可能是因为 Jest 已经 **缓存** 了构建结果。要应用新配置，你需要清除 Jest 的缓存目录。

要清除缓存目录，请在 NestJS 项目文件夹中运行以下命令：

```bash
$ npx jest --clearCache
```

如果自动缓存清除失败，你仍然可以手动删除缓存文件夹，使用以下命令：

```bash
# 查找 jest 缓存目录（通常是 /tmp/jest_rs）
# 通过在 NestJS 项目根目录运行以下命令
$ npx jest --showConfig | grep cache
# 示例结果：
#   "cache": true,
#   "cacheDirectory": "/tmp/jest_rs"

# 删除或清空 Jest 缓存目录
$ rm -rf  <cacheDirectory value>
# 示例：
# rm -rf /tmp/jest_rs
```