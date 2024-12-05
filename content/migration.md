### 迁移指南

本文提供了一套从Nest版本9迁移到版本10的指导方针。
要了解更多关于我们在v10中添加的新功能，请查看此[文章](https://trilon.io/blog/nestjs-10-is-now-available)。
有一些非常小的破坏性变更，应该不会影响大多数用户 - 你可以在这里找到它们的完整列表[这里](https://github.com/nestjs/nest/releases/tag/v10.0.0)。

#### 升级包

虽然你可以手动升级你的包，但我们推荐使用[ncu (npm检查更新)](https://npmjs.com/package/npm-check-updates)。

#### 缓存模块

`CacheModule`已从`@nestjs/common`包中移除，现在作为一个独立的包提供 - `@nestjs/cache-manager`。这个变更是为了避免在`@nestjs/common`包中不必要的依赖。你可以在这里了解更多关于`@nestjs/cache-manager`包的信息[这里](https://docs.nestjs.com/techniques/caching)。

#### 弃用

所有已弃用的方法和模块已被移除。

#### CLI插件和TypeScript >= 4.8

NestJS CLI插件（适用于`@nestjs/swagger`和`@nestjs/graphql`包）现在将需要TypeScript >= v4.8，因此不再支持旧版本的TypeScript。这个变更的原因是，在[TypeScript v4.8引入了其抽象语法树（AST）的几个破坏性变更](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-8.html#decorators-are-placed-on-modifiers-on-typescripts-syntax-trees)，我们使用它来自动生成OpenAPI和GraphQL模式。

#### 停止支持Node.js v12

从NestJS 10开始，我们不再支持Node.js v12，因为[v12在2022年4月30日结束了生命周期](https://twitter.com/nodejs/status/1524081123579596800)。这意味着NestJS 10需要Node.js v16或更高版本。这个决定使我们最终可以在TypeScript配置中设置目标为`ES2021`，而不是像过去那样提供polyfills。

从现在开始，每个官方NestJS包将默认编译为`ES2021`，这应该会得到更小的库大小，有时甚至（略微）更好的性能。

我们还强烈推荐使用最新的LTS版本。