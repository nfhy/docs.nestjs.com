### 异步本地存储

`AsyncLocalStorage` 是一个 [Node.js API](https://nodejs.org/api/async_context.html#async_context_class_asynclocalstorage)（基于 `async_hooks` API），它提供了一种在应用程序中传播本地状态的替代方式，无需将其作为函数参数显式传递。它类似于其他语言中的线程本地存储。

异步本地存储的主要思想是，我们可以用 `AsyncLocalStorage#run` 调用来“包装”某些函数调用。所有在包装调用中被调用的代码都可以访问同一个 `store`，这个 `store` 对于每个调用链都是唯一的。

在 NestJS 的上下文中，这意味着如果我们可以在请求生命周期的某个地方包装其余的请求代码，我们将能够访问和修改仅对该请求可见的状态，这可能是对请求范围提供者及其一些限制的替代方案。

或者，我们可以使用 ALS 仅为系统的某一部分（例如事务对象）传播上下文，而无需在服务之间显式传递，这可以增加隔离和封装。

#### 自定义实现

NestJS 本身没有提供任何内置的 `AsyncLocalStorage` 抽象，因此让我们逐步了解如何为最简单的 HTTP 案例实现它，以更好地理解整个概念：

> **信息** 对于现成的 [专用包](recipes/async-local-storage#nestjs-cls)，请继续阅读以下内容。

1. 首先，在某个共享源文件中创建一个新的 `AsyncLocalStorage` 实例。由于我们使用的是 NestJS，让我们也将其变成一个模块，并提供一个自定义提供者。

```ts
@@filename(als.module)
@Module({
  providers: [
    {
      provide: AsyncLocalStorage,
      useValue: new AsyncLocalStorage(),
    },
  ],
  exports: [AsyncLocalStorage],
})
export class AlsModule {}
```

> **提示** `AsyncLocalStorage` 是从 `async_hooks` 导入的。

2. 我们只关心 HTTP，因此让我们使用中间件来用 `AsyncLocalStorage#run` 包装 `next` 函数。由于中间件是请求首先击中的东西，这将使 `store` 在所有增强器和系统的其余部分中可用。

```ts
@@filename(app.module)
@Module({
  imports: [AlsModule]
  providers: [CatService],
  controllers: [CatController],
})
export class AppModule implements NestModule {
  constructor(
    // 在模块构造函数中注入 AsyncLocalStorage，
    private readonly als: AsyncLocalStorage
  ) {}

  configure(consumer: MiddlewareConsumer) {
    // 绑定中间件，
    consumer
      .apply((req, res, next) => {
        // 根据请求填充存储的一些默认值，
        const store = {
          userId: req.headers['x-user-id'],
        };
        // 并将“next”函数作为回调传递给“als.run”方法以及存储。
        this.als.run(store, () => next());
      })
      // 并为所有路由注册（如果使用 Fastify，则使用 '(.*)'）
      .forRoutes('*');
  }
}
@@switch
@Module({
  imports: [AlsModule]
  providers: [CatService],
  controllers: [CatController],
})
@Dependencies(AsyncLocalStorage)
export class AppModule {
  constructor(als) {
    // 在模块构造函数中注入 AsyncLocalStorage，
    this.als = als
  }

  configure(consumer) {
    // 绑定中间件，
    consumer
      .apply((req, res, next) => {
        // 根据请求填充存储的一些默认值，
        const store = {
          userId: req.headers['x-user-id'],
        };
        // 并将“next”函数作为回调传递给“als.run”方法以及存储。
        this.als.run(store, () => next());
      })
      // 并为所有路由注册（如果使用 Fastify，则使用 '(.*)'）
      .forRoutes('*');
  }
}
```

3. 现在，在请求的生命周期中的任何地方，我们都可以访问本地存储实例。

```ts
@@filename(cat.service)
@Injectable()
export class CatService {
  constructor(
    // 我们可以注入提供的 ALS 实例。
    private readonly als: AsyncLocalStorage,
    private readonly catRepository: CatRepository,
  ) {}

  getCatForUser() {
    // “getStore”方法将始终返回与给定请求关联的存储实例。
    const userId = this.als.getStore()["userId"] as number;
    return this.catRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatRepository)
export class CatService {
  constructor(als, catRepository) {
    // 我们可以注入提供的 ALS 实例。
    this.als = als
    this.catRepository = catRepository
  }

  getCatForUser() {
    // “getStore”方法将始终返回与给定请求关联的存储实例。
    const userId = this.als.getStore()["userId"] as number;
    return this.catRepository.getForUser(userId);
  }
}
```

4. 就是这样。现在我们有了一种在不需要注入整个 `REQUEST` 对象的情况下共享与请求相关的州的方法。

> **警告** 请注意，虽然这种技术对许多用例都很有用，但它本质上混淆了代码流程（创建隐式上下文），因此请负责任地使用，特别是避免创建上下文的“[上帝对象](https://en.wikipedia.org/wiki/God_object)”。

### NestJS CLS

[nestjs-cls](https://github.com/Papooch/nestjs-cls) 包提供了几个 DX 改进，超过了使用纯 `AsyncLocalStorage`（CLS 是术语 _continuation-local storage_ 的缩写）。它将实现抽象到一个 `ClsModule` 中，提供了多种初始化不同传输方式（不仅仅是 HTTP）的 `store` 的方法，以及强类型支持。

然后可以使用可注入的 `ClsService` 访问存储值，或者通过使用 [Proxy Providers](https://www.npmjs.com/package/nestjs-cls#proxy-providers) 完全从业务逻辑中抽象出来。

> **信息** `nestjs-cls` 是第三方包，不是由 NestJS 核心团队管理的。请在 [适当的仓库](https://github.com/Papooch/nestjs-cls/issues) 中报告任何发现的问题。

#### 安装

除了对 `@nestjs` 库的对等依赖外，它只使用内置的 Node.js API。像安装任何其他包一样安装它。

```bash
npm i nestjs-cls
```

#### 使用

可以使用 `nestjs-cls` 实现类似于 [上述](recipes/async-local-storage#custom-implementation) 的功能，如下所示：

1. 在根模块中导入 `ClsModule`。

```ts
@@filename(app.module)
@Module({
  imports: [
    // 注册 ClsModule，
    ClsModule.forRoot({
      middleware: {
        // 自动为所有路由安装
        // ClsMiddleware
        mount: true,
        // 并使用 setup 方法提供默认存储值。
        setup: (cls, req) => {
          cls.set('userId', req.headers['x-user-id']);
        },
      },
    }),
  ],
  providers: [CatService],
  controllers: [CatController],
})
export class AppModule {}
```

2. 然后可以使用 `ClsService` 访问存储值。

```ts
@@filename(cat.service)
@Injectable()
export class CatService {
  constructor(
    // 我们可以注入提供的 ClsService 实例，
    private readonly cls: ClsService,
    private readonly catRepository: CatRepository,
  ) {}

  getCatForUser() {
    // 并使用 “get” 方法检索任何存储的值。
    const userId = this.cls.get('userId');
    return this.catRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatRepository)
export class CatService {
  constructor(cls, catRepository) {
    // 我们可以注入提供的 ClsService 实例，
    this.cls = cls
    this.catRepository = catRepository
  }

  getCatForUser() {
    // 并使用 “get” 方法检索任何存储的值。
    const userId = this.cls.get('userId');
    return this.catRepository.getForUser(userId);
  }
}
```

3. 要获得由 `ClsService` 管理的存储值的强类型（还可以获得字符串键的自动建议），我们可以在注入时使用可选的类型参数 `ClsService<MyClsStore>`。

```ts
export interface MyClsStore extends ClsStore {
  userId: number;
}
```

> **提示** 还可以让包自动生成请求 ID，并稍后使用 `cls.getId()` 访问，或者使用 `cls.get(CLS_REQ)` 获取整个请求对象。

#### 测试

由于 `ClsService` 只是另一个可注入的提供者，它可以在单元测试中完全被模拟。

然而，在某些集成测试中，我们可能仍然想要使用真实的 `ClsService` 实现。在这种情况下，我们需要用 `ClsService#run` 或 `ClsService#runWith` 的调用来包装上下文感知的代码片段。

```ts
describe('CatService', () => {
  let service: CatService
  let cls: ClsService
  const mockCatRepository = createMock<CatRepository>()

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      // 像我们通常做的那样设置大部分测试模块。
      providers: [
        CatService,
        {
          provide: CatRepository
          useValue: mockCatRepository
        }
      ],
      imports: [
        // 导入 ClsModule 的静态版本，它只提供
        // ClsService，但以任何方式设置存储。
        ClsModule
      ],
    }).compile()

    service = module.get(CatService)

    // 也检索 ClsService 以备后用。
    cls = module.get(ClsService)
  })

  describe('getCatForUser', () => {
    it('根据用户 ID 检索猫', async () => {
      const expectedUserId = 42
      mockCatRepository.getForUser.mockImplementationOnce(
        (id) => ({ userId: id })
      )

      // 在 `runWith` 方法中包装测试调用
      // 在我们可以传递手工制作的存储值的地方。
      const cat = await cls.runWith(
        { userId: expectedUserId },
        () => service.getCatForUser()
      )

      expect(cat.userId).toEqual(expectedUserId)
    })
  })
})
```

#### 更多信息

访问 [NestJS CLS GitHub 页面](https://github.com/Papooch/nestjs-cls) 以获取完整的 API 文档和更多代码示例。