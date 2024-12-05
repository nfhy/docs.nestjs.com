### 常见错误

在使用NestJS进行开发时，您可能会遇到各种错误，因为您在学习这个框架。

#### “无法解析依赖”错误

> 信息 **提示** 查看 [NestJS Devtools](/devtools/overview#investigating-the-cannot-resolve-dependency-error)，它可以帮助您轻松解决“无法解析依赖”错误。

最常见的错误信息是关于Nest无法解析某个提供者的依赖。错误消息通常看起来像这样：

```bash
Nest无法解析<provider>的依赖。请确保参数<unknown_token>在<module>上下文中的索引[<index>]是可用的。

潜在解决方案：
- <module>是一个有效的NestJS模块吗？
- 如果<unknown_token>是一个提供者，它是当前<module>的一部分吗？
- 如果<unknown_token>从单独的@Module导出，那个模块是否在<module>中导入？
  @Module({
    imports: [ /* 包含<unknown_token>的Module */ ]
  })
```

这个错误最常见的原因是没有将`<provider>`放在模块的`providers`数组中。请确保提供者确实在`providers`数组中，并遵循[标准的NestJS提供者实践](/fundamentals/custom-providers#di-fundamentals)。

有一些常见的陷阱。一个是将提供者放在`imports`数组中。如果这样的话，错误将把提供者的名称放在`<module>`应该在的地方。

如果您在开发中遇到这个错误，请查看错误消息中提到的模块及其`providers`。对于`providers`数组中的每个提供者，确保模块可以访问所有依赖。通常，`providers`会在“特性模块”和“根模块”中重复，这意味着Nest将尝试两次实例化提供者。最有可能的是，包含被重复的`<provider>`的模块应该被添加到“根模块”的`imports`数组中。

如果上述`<unknown_token>`是`dependency`，您可能有循环文件导入。这与下面的[循环依赖](/faq/common-errors#circular-dependency-error)不同，因为不是提供者在它们的构造函数中相互依赖，只是意味着两个文件最终相互导入。一个常见的情况是模块文件声明一个令牌并导入一个提供者，提供者从模块文件导入令牌常量。如果您使用桶文件，请确保您的桶导入不会最终创建这些循环导入。

如果上述`<unknown_token>`是`Object`，这意味着您在使用类型/接口进行注入，而没有适当的提供者令牌。要解决这个问题，请确保：

1. 您正在导入类引用或使用带有`@Inject()`装饰器的自定义令牌。阅读[自定义提供者页面](/fundamentals/custom-providers)，
2. 对于基于类的提供者，您正在导入具体的类，而不仅仅是通过[`import type ...`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-8.html#type-only-imports-and-export)语法导入类型。

同时，请确保您没有最终注入提供者本身，因为在NestJS中不允许自我注入。当这种情况发生时，`<unknown_token>`可能等于`<provider>`。

<app-banner-devtools></app-banner-devtools>

如果您在**单仓库设置**中，您可能会遇到上述相同的错误，但是针对核心提供者`ModuleRef`作为`<unknown_token>`：

```bash
Nest无法解析<provider>的依赖。
请确保参数ModuleRef在<module>上下文中的索引[<index>]是可用的。
...
```

这很可能发生在您的项目最终加载了两个`@nestjs/core`包的Node模块，如下所示：

```text
.
├── package.json
├── apps
│   └── api
│       └── node_modules
│           └── @nestjs/bull
│               └── node_modules
│                   └── @nestjs/core
└── node_modules
    ├── (其他包)
    └── @nestjs/core
```

解决方案：

- 对于**Yarn** Workspaces，使用[nohoist特性](https://classic.yarnpkg.com/blog/2018/02/15/nohoist)以防止提升`@nestjs/core`包。
- 对于**pnpm** Workspaces，在您的其他模块中将`@nestjs/core`设置为peerDependencies，并在导入模块的app package.json中设置`"dependenciesMeta": {"other-module-name": {"injected": true}}`。参见：[dependenciesmetainjected](https://pnpm.io/package_json#dependenciesmetainjected)

#### “循环依赖”错误

偶尔，您会发现在您的应用程序中很难避免[循环依赖](https://docs.nestjs.com/fundamentals/circular-dependency)。您需要采取一些步骤来帮助Nest解决这些问题。由循环依赖引起的错误看起来像这样：

```bash
Nest无法创建<module>实例。
<module>的“imports”数组中索引[<index>]处的模块是未定义的。

潜在原因：
- 模块之间的循环依赖。使用forwardRef()来避免它。阅读更多：https://docs.nestjs.com/fundamentals/circular-dependency
- 索引[<index>]处的模块类型是“undefined”。检查您的导入语句和模块的类型。

作用域[<module_import_chain>]
# 示例链 AppModule -> FooModule
```

循环依赖可能源于提供者相互依赖，或者TypeScript文件相互依赖以获取常量，例如从模块文件中导出常量并在服务文件中导入它们。在后一种情况下，建议为您的常量创建一个单独的文件。在前一种情况下，请按照循环依赖的指南操作，并确保两个模块**和**提供者都标记为`forwardRef`。

#### 调试依赖错误

除了手动验证您的依赖关系是否正确之外，从Nest 8.1.0开始，您可以设置`NEST_DEBUG`环境变量为一个解析为真值的字符串，并在Nest解析应用程序的所有依赖关系时获得额外的日志信息。

<figure><img src="/assets/injector_logs.png" /></figure>

在上面的图片中，黄色字符串是被注入依赖的宿主类，蓝色字符串是注入的依赖项的名称或其注入令牌，紫色字符串是正在搜索依赖项的模块。利用这些，您通常可以追溯回依赖解析发生了什么以及为什么您会遇到依赖注入问题。

#### “文件更改检测”循环无尽

使用TypeScript版本4.9及以上的Windows用户可能会遇到这个问题。

当您尝试以监视模式运行应用程序时，例如`npm run start:dev`，您会看到日志消息的无尽循环：

```bash
XX:XX:XX 上午 - 检测到文件更改。开始增量编译...
XX:XX:XX 上午 - 未发现错误。正在监视文件更改。
```

当您使用NestJS CLI以监视模式启动应用程序时，这是通过调用`tsc --watch`完成的，而从TypeScript版本4.9开始，[用于检测文件更改的新策略](https://devblogs.microsoft.com/typescript/announcing-typescript-4-9/#file-watching-now-uses-file-system-events)可能是导致这个问题的原因。

为了解决这个问题，您需要在`tsconfig.json`文件中，在`"compilerOptions"`选项之后添加一个设置，如下所示：

```bash
  "watchOptions": {
    "watchFile": "fixedPollingInterval"
  }
```

这告诉TypeScript使用轮询方法来检查文件更改，而不是文件系统事件（新默认方法），这可能会在某些机器上引起问题。

您可以在[TypeScript文档](https://www.typescriptlang.org/tsconfig#watch-watchDirectory)中了解更多关于`"watchFile"`选项的信息。