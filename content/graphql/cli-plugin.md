### CLI 插件

> 警告 **警告** 本章节仅适用于代码优先方法。

TypeScript 的元数据反射系统有一些限制，这使得它无法确定一个类包含哪些属性，或者识别给定属性是可选的还是必需的。然而，这些限制中的一些可以在编译时解决。Nest 提供了一个插件，增强了 TypeScript 编译过程，以减少所需的样板代码量。

> 提示 **提示** 此插件是 **可选的**。如果您喜欢，可以手动声明所有装饰器，或者只在需要的地方声明特定的装饰器。

#### 概览

GraphQL 插件将自动执行以下操作：

- 除非使用了 `@HideField`，否则为所有输入对象、对象类型和参数类属性添加 `@Field` 注解
- 根据问号设置 `nullable` 属性（例如 `name?: string` 将设置 `nullable: true`）
- 根据类型设置 `type` 属性（支持数组）
- 如果 `introspectComments` 设置为 `true`，则基于注释生成属性描述

请注意，您的文件名**必须**有以下后缀之一，以便被插件分析：`['.input.ts', '.args.ts', '.entity.ts', '.model.ts']`（例如，`author.entity.ts`）。如果您使用不同的后缀，可以通过指定 `typeFileNameSuffix` 选项来调整插件的行为（见下文）。

根据我们目前所学的，您需要重复很多代码，以让包知道如何在 GraphQL 中声明您的类型。例如，您可以如下定义一个简单的 `Author` 类：

```typescript
@@filename(authors/models/author.model)
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

虽然对于中等规模的项目来说这不是一个显著的问题，但一旦您拥有大量的类，就会变得冗长且难以维护。

通过启用 GraphQL 插件，上述类定义可以简单地声明为：

```typescript
@@filename(authors/models/author.model)
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;
  firstName?: string;
  lastName?: string;
  posts: Post[];
}
```

插件会根据 **抽象语法树** 动态添加适当的装饰器。因此，您不必在代码中到处挣扎着使用 `@Field` 装饰器。

> 提示 **提示** 插件会自动生成任何缺失的 GraphQL 属性，但如果您需要覆盖它们，只需通过 `@Field()` 明确设置即可。

#### 注释内省

启用注释内省功能后，CLI 插件将根据注释为字段生成描述。

例如，给定一个 `roles` 属性的例子：

```typescript
/**
 * 用户的角色列表
 */
@Field(() => [String], {
  description: `用户的角色列表`
})
roles: string[];
```

您必须重复描述值。启用 `introspectComments` 后，CLI 插件可以提取这些注释，并自动为属性提供描述。现在，上述字段可以简单地声明如下：

```typescript
/**
 * 用户的角色列表
 */
roles: string[];
```

#### 使用 CLI 插件

要启用插件，请打开 `nest-cli.json`（如果您使用 [Nest CLI](/cli/overview)）并添加以下 `plugins` 配置：

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/graphql"]
  }
}
```

您可以使用 `options` 属性来自定义插件的行为。

```javascript
"plugins": [
  {
    "name": "@nestjs/graphql",
    "options": {
      "typeFileNameSuffix": [".input.ts", ".args.ts"],
      "introspectComments": true
    }
  }
]
```

`options` 属性必须满足以下接口：

```typescript
export interface PluginOptions {
  typeFileNameSuffix?: string[];
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
    <td><code>typeFileNameSuffix</code></td>
    <td><code>['.input.ts', '.args.ts', '.entity.ts', '.model.ts']</code></td>
    <td>GraphQL 类型文件后缀</td>
  </tr>
  <tr>
    <td><code>introspectComments</code></td>
    <td><code>false</code></td>
    <td>如果设置为 true，则插件将根据注释生成属性描述</td>
  </tr>
</table>

如果您不使用 CLI，而是有一个自定义的 `webpack` 配置，您可以将此插件与 `ts-loader` 结合使用：

```javascript
getCustomTransformers: (program: any) => ({
  before: [require('@nestjs/graphql/plugin').before({}, program)]
}),
```

#### SWC 构建器

对于标准设置（非 monorepo），要使用 CLI 插件与 SWC 构建器，您需要启用类型检查，如 [此处](/recipes/swc#type-checking) 所述。

```bash
$ nest start -b swc --type-check
```

对于 monorepo 设置，请按照 [此处](/recipes/swc#monorepo-and-cli-plugins) 的说明操作。

```bash
$ npx ts-node src/generate-metadata.ts
# OR npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

现在，序列化的元数据文件必须由 `GraphQLModule` 方法加载，如下所示：

```typescript
import metadata from './metadata'; // <-- 文件由 "PluginMetadataGenerator" 自动生成

GraphQLModule.forRoot<...>({
  ..., // 其他选项
  metadata,
}),
```

#### 与 `ts-jest`（端到端测试）集成

当启用此插件运行端到端测试时，您可能会遇到编译模式的问题。例如，最常见的错误之一是：

```json
Object type <name> must define one or more fields.
```

这是因为 `jest` 配置没有在任何地方导入 `@nestjs/graphql/plugin` 插件。

要解决这个问题，请在您的端到端测试目录中创建以下文件：

```javascript
const transformer = require('@nestjs/graphql/plugin');

module.exports.name = 'nestjs-graphql-transformer';
// 您应该在更改下面的配置时更改版本号 - 否则，jest 将不会检测到更改
module.exports.version = 1;

module.exports.factory = (cs) => {
  return transformer.before(
    {
      // @nestjs/graphql/plugin 选项（可以为空）
    },
    cs.program, // 对于旧版本的 Jest（<= v27），使用 "cs.tsCompiler.program"
  );
};
```

有了这个，就在您的 `jest` 配置文件中导入 AST 转换器。默认情况下（在起始应用程序中），端到端测试配置文件位于 `test` 文件夹中，名为 `jest-e2e.json`。

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

如果您使用 `jest@^29`，则使用下面的代码片段，因为前面的方法是被弃用的。

```json
{
  ... // 其他配置
  "transform": {
    "^.+\\.(t|j)s$": [
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