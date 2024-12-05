循环依赖

循环依赖发生在两个类相互依赖时。例如，类A需要类B，而类B也需要类A。循环依赖可能在Nest中出现在模块之间和提供者之间。

虽然应尽可能避免循环依赖，但有时无法避免。在这种情况下，Nest允许通过两种方式解决提供者之间的循环依赖。在本章中，我们描述了使用**向前引用**作为一项技术，以及使用**ModuleRef**类从DI容器中检索提供者实例作为另一种技术。

我们还描述了解决模块之间的循环依赖。

> 警告 **警告** 当使用“桶文件”/index.ts文件来分组导入时，也可能导致循环依赖。桶文件在涉及模块/提供者类时应省略。例如，桶文件不应在导入与桶文件同一目录下的文件时使用，即 `cats/cats.controller` 不应导入 `cats` 以导入 `cats/cats.service` 文件。更多详情请参阅[这个GitHub问题](https://github.com/nestjs/nest/issues/1181#issuecomment-430197191)。

#### 向前引用

**向前引用**允许Nest引用尚未定义的类，使用`forwardRef()`实用函数。例如，如果`CatsService`和`CommonService`相互依赖，关系的双方都可以使用`@Inject()`和`forwardRef()`实用函数来解决循环依赖。否则Nest不会实例化它们，因为所有必要的元数据将不可用。这里有一个例子：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private commonService: CommonService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CommonService))
export class CatsService {
  constructor(commonService) {
    this.commonService = commonService;
  }
}
```

> 提示 **提示** `forwardRef()`函数从`@nestjs/common`包导入。

这涵盖了关系的一边。现在让我们对`CommonService`做同样的处理：

```typescript
@@filename(common.service)
@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private catsService: CatsService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CatsService))
export class CommonService {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

> 警告 **警告** 实例化顺序是不确定的。确保你的代码不依赖于哪个构造函数先被调用。具有`Scope.REQUEST`的循环依赖可能导致未定义的依赖。更多信息可在此查看[这里](https://github.com/nestjs/nest/issues/5778)。

#### ModuleRef类替代方案

使用`forwardRef()`的替代方案是重构代码，使用`ModuleRef`类来检索（原本）循环关系一侧的提供者。了解更多关于`ModuleRef`实用类的[这里](/fundamentals/module-ref)。

#### 模块向前引用

为了解决模块之间的循环依赖，使用相同的`forwardRef()`实用函数在模块关联的两边。例如：

```typescript
@@filename(common.module)
@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```

这涵盖了关系的一边。现在让我们对`CatsModule`做同样的处理：

```typescript
@@filename(cats.module)
@Module({
  imports: [forwardRef(() => CommonModule)],
})
export class CatsModule {}
```