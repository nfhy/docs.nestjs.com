自动化测试被认为是任何严肃的软件开发工作的重要组成部分。自动化使得在开发过程中快速且轻松地重复单个测试或测试套件成为可能。这有助于确保发布版本满足质量和性能目标。自动化提高了覆盖率，并为开发人员提供了更快的反馈循环。自动化不仅提高了个别开发人员的生产力，还确保了在关键的开发生命周期节点运行测试，例如源代码控制检入、功能集成和版本发布。

这样的测试通常包括多种类型，包括单元测试、端到端（e2e）测试、集成测试等。虽然好处是毫无疑问的，但设置它们可能会很繁琐。Nest 致力于推广开发最佳实践，包括有效的测试，因此它包括以下功能来帮助开发人员和团队构建和自动化测试：

- 自动为组件生成默认单元测试，为应用程序生成默认的e2e测试
- 提供默认工具（例如构建隔离模块/应用程序加载器的测试运行器）
- 提供与 [Jest](https://github.com/facebook/jest) 和 [Supertest](https://github.com/visionmedia/supertest) 的即插即用集成，同时对测试工具保持中立
- 使得Nest依赖注入系统在测试环境中可用，便于模拟组件

如上所述，您可以使用任何您喜欢的**测试框架**，因为Nest不会强制使用任何特定的工具。简单地替换所需的元素（例如测试运行器），您仍然可以享受Nest现成的测试设施的好处。

#### 安装

要开始使用，请先安装所需的包：

```bash
$ npm i --save-dev @nestjs/testing
```

#### 单元测试

在以下示例中，我们测试两个类：`CatsController` 和 `CatsService`。如上所述，[Jest](https://github.com/facebook/jest) 作为默认的测试框架提供。它充当测试运行器，并且还提供断言函数和测试双实用程序，帮助进行模拟、间谍等。在以下基本测试中，我们手动实例化这些类，并确保控制器和服务履行它们的API契约。

```typescript
@@filename(cats.controller.spec)
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

> 信息 **提示** 将测试文件存放在它们测试的类附近。测试文件应有 `.spec` 或 `.test` 后缀。

由于上述示例是微不足道的，我们实际上并没有测试任何Nest特定的内容。实际上，我们甚至没有使用依赖注入（注意到我们将 `CatsService` 的实例传递给我们的 `catsController`）。这种测试形式 - 我们手动实例化被测试的类 - 通常被称为**隔离测试**，因为它与框架无关。让我们引入一些更高级的功能，帮助您测试更广泛使用Nest特性的应用程序。

#### 测试工具

`@nestjs/testing` 包提供了一组实用工具，使测试过程更加健壮。让我们使用内置的 `Test` 类重写前面的示例：

```typescript
@@filename(cats.controller.spec)
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get(CatsService);
    catsController = moduleRef.get(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get(CatsService);
    catsController = moduleRef.get(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

`Test` 类很有用，因为它提供了一个应用程序执行上下文，基本上模拟了完整的Nest运行时，但是提供了使管理类实例（包括模拟和覆盖）变得简单的钩子。`Test` 类有一个 `createTestingModule()` 方法，它接受模块元数据对象作为其参数（与您传递给 `@Module()` 装饰器的相同对象）。此方法返回一个 `TestingModule` 实例，该实例反过来提供几种方法。对于单元测试，重要的是 `compile()` 方法。此方法引导一个模块及其依赖项（类似于在传统的 `main.ts` 文件中使用 `NestFactory.create()` 引导应用程序的方式），并返回一个准备好进行测试的模块。

> 信息 **提示** `compile()` 方法是 **异步的**，因此必须等待。一旦模块编译完成，您可以使用 `get()` 方法检索它声明的任何 **静态** 实例（控制器和提供者）。

`TestingModule` 继承自 [模块引用](/fundamentals/module-ref) 类，因此它能够动态解析作用域提供者（临时或请求作用域）。使用 `resolve()` 方法进行此操作（`get()` 方法只能检索静态实例）。

```typescript
const moduleRef = await Test.createTestingModule({
  controllers: [CatsController],
  providers: [CatsService],
}).compile();

catsService = await moduleRef.resolve(CatsService);
```

> 警告 **警告** `resolve()` 方法返回提供者的唯一实例，来自其自己的 **DI容器子树**。每个子树都有唯一的上下文标识符。因此，如果您多次调用此方法并比较实例引用，您会发现它们不相等。

> 信息 **提示** 在 [这里](/fundamentals/module-ref) 了解更多关于模块引用特性的信息。

而不是使用任何提供者的生产版本，您可以为测试目的覆盖它与 [自定义提供者](/fundamentals/custom-providers)。例如，您可以模拟数据库服务而不是连接到实时数据库。我们将在下一节中介绍覆盖，但它们也适用于单元测试。

<app-banner-courses></app-banner-courses>

#### 自动模拟

Nest还允许您定义一个模拟工厂，适用于您所有缺失的依赖项。这在您有一个类中有很多依赖项，并且模拟它们全部将花费很长时间和大量设置的情况下非常有用。要使用此功能，`createTestingModule()` 需要与 `useMocker()` 方法链式调用，传递一个工厂用于您的依赖项模拟。这个工厂可以接收一个可选的令牌，这是一个实例令牌，任何对于Nest提供者有效的令牌，并返回一个模拟实现。以下是使用 [`jest-mock`](https://www.npmjs.com/package/jest-mock) 创建通用模拟器和使用 `jest.fn()` 为 `CatsService` 创建特定模拟的示例。

```typescript
// ...
import { ModuleMocker, MockFunctionMetadata } from 'jest-mock';

const moduleMocker = new ModuleMocker(global);

describe('CatsController', () => {
  let controller: CatsController;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [CatsController],
    })
      .useMocker((token) => {
        const results = ['test1', 'test2'];
        if (token === CatsService) {
          return { findAll: jest.fn().mockResolvedValue(results) };
        }
        if (typeof token === 'function') {
          const mockMetadata = moduleMocker.getMetadata(
            token,
          ) as MockFunctionMetadata<any, any>;
          const Mock = moduleMocker.generateFromMetadata(mockMetadata);
          return new Mock();
        }
      })
      .compile();

    controller = moduleRef.get(CatsController);
  });
});
```

您也可以像您通常会自定义提供者一样从测试容器中检索这些模拟，`moduleRef.get(CatsService)`。

> 信息 **提示** 一个通用模拟工厂，如 `createMock` 来自 [`@golevelup/ts-jest`](https://github.com/golevelup/nestjs/tree/master/packages/testing) 也可以直接传递。

> 信息 **提示** `REQUEST` 和 `INQUIRER` 提供者不能自动模拟，因为它们已经在上下文中预定义。然而，它们可以使用自定义提供者语法或利用 `.overrideProvider` 方法被 **覆盖**。

#### 端到端测试

与单元测试不同，单元测试侧重于单独的模块和类，端到端（e2e）测试涵盖了类和模块在更聚合层面上的交互 - 更接近于最终用户将与生产系统进行的交互。随着应用程序的增长，手动测试每个API端点的端到端行为变得越来越困难。自动化端到端测试帮助我们确保系统的整体行为是正确的，并且符合项目要求。要执行e2e测试，我们使用与我们在 **单元测试** 中刚刚介绍的类似配置。此外，Nest使您能够轻松使用 [Supertest](https://github.com/visionmedia/supertest) 库来模拟HTTP请求。

```typescript
@@filename(cats.e2e-spec)
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
@@switch
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

> 信息 **提示** 如果您使用 [Fastify](/techniques/performance) 作为您的HTTP适配器，它需要稍微不同的配置，并且具有内置的测试功能：
>

```ts
let app: NestFastifyApplication;

beforeAll(async () => {
  app = moduleRef.createNestApplication<NestFastifyApplication>(
    new FastifyAdapter(),
  );

  await app.init();
  await app.getHttpAdapter().getInstance().ready();
});

it(`/GET cats`, () => {
  return app
    .inject({
      method: 'GET',
      url: '/cats',
    })
    .then((result) => {
      expect(result.statusCode).toEqual(200);
      expect(result.payload).toEqual(/* expectedPayload */);
    });
});

afterAll(async () => {
  await app.close();
});
```

在这个示例中，我们建立在前面描述的一些概念之上。除了我们之前使用的 `compile()` 方法，我们现在使用 `createNestApplication()` 方法来实例化一个完整的Nest运行时环境。

需要注意的一个警告是，当您的应用程序使用 `compile()` 方法编译时，`HttpAdapterHost#httpAdapter` 在那时将是未定义的。这是因为在编译阶段还没有创建HTTP适配器或服务器。如果您的测试需要 `httpAdapter`，您应该使用 `createNestApplication()` 方法来创建应用程序实例，或者重构您的项目，以避免在初始化依赖图时依赖于此。

好的，让我们分解这个示例：

我们在 `app` 变量中保存对运行中的应用程序的引用，以便我们可以使用它来模拟HTTP请求。

我们使用Supertest的 `request()` 函数来模拟HTTP测试。我们希望这些HTTP请求路由到我们的运行中的Nest应用程序，因此我们将 `request()` 函数传递一个引用到底层Nest的HTTP监听器（这反过来可能由Express平台提供）。因此构造 `request(app.getHttpServer())`。调用 `request()` 将给我们一个包装的HTTP服务器，现在连接到Nest应用程序，它暴露了模拟实际HTTP请求的方法。例如，使用 `request(...).get('/cats')` 将启动一个与通过网络传入的 **实际** HTTP请求 `get '/cats'` 相同的请求到Nest应用程序。

在这个示例中，我们还提供了 `CatsService` 的替代（测试双）实现，它简单地返回一个我们可以测试的硬编码值。使用 `overrideProvider()` 提供这样的替代实现。类似地，Nest提供了方法来覆盖模块、守卫、拦截器、过滤器和管道，分别使用 `overrideModule()`、`overrideGuard()`、`overrideInterceptor()`、`overrideFilter()` 和 `overridePipe()` 方法。

另一方面，`overrideModule()` 返回一个对象，该对象具有 `useModule()` 方法，您可以使用它来提供一个将覆盖原始模块的模块，如下所示：

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideModule(CatsModule)
  .useModule(AlternateCatsModule)
  .compile();
```

每个覆盖方法类型，反过来，返回 `TestingModule` 实例，因此可以用其他方法以 [流畅风格](https://en.wikipedia.org/wiki/Fluent_interface) 链式调用。您应该在这样的链的末尾使用 `compile()` 来使Nest实例化和初始化模块。

此外，有时您可能想要在测试运行时（例如，在CI服务器上）提供自定义日志记录器。使用 `setLogger()` 方法并传递一个满足 `LoggerService` 接口的对象，以指示 `TestModuleBuilder` 在测试期间如何记录（默认情况下，只有 "error" 日志会记录到控制台）。

编译后的模块有几个有用的方法，如下表所述：

<table>
  <tr>
    <td>
      <code>createNestApplication()</code>
    </td>
    <td>
      基于给定模块创建并返回一个Nest应用程序（<code>INestApplication</code>实例）。注意，您必须手动使用 <code>init()</code> 方法初始化应用程序。
    </td>
  </tr>
  <tr>
    <td>
      <code>createNestMicroservice()</code>
    </td>
    <td>
      基于给定模块创建并返回一个Nest微服务（<code>INestMicroservice</code>实例）。
    </td>
  </tr>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      检索应用程序上下文中可用的控制器或提供者（包括守卫、过滤器等）的静态实例。从 <a href="/fundamentals/module-ref">模块引用</a> 类继承。
    </td>
  </tr>
  <tr>
    <td>
      <code>resolve()</code>
    </td>
    <td>
      检索应用程序上下文中可用的动态创建的作用域实例（请求或瞬态）的控制器或提供者（包括守卫、过滤器等）。从 <a href="/fundamentals/module-ref">模块引用</a> 类继承。
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      浏览模块的依赖图；可用于从所选模块检索特定实例（与严格模式（<code>strict: true</code>）一起在 <code>get()</code> 方法中使用）。
    </td>
  </tr>
</table>

> 信息 **提示** 将您的e2e测试文件放在 `test` 目录中。测试文件应有 `.e2e-spec` 后缀。

#### 覆盖全局注册的增强器

如果您有全局注册的守卫（或管道、拦截器、过滤器），您需要采取更多步骤来覆盖该增强器。回顾原始注册如下：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

这是通过 `APP_*` 令牌注册守卫作为 "多" 提供者的。

要能够替换这里的 `JwtAuthGuard`，注册需要使用此插槽中的现有提供者：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useExisting: JwtAuthGuard,
    // ^^^^^^^^ 注意使用 'useExisting' 而不是 'useClass'
  },
  JwtAuthGuard,
],
```

> 信息 **提示** 将 `useClass` 更改为 `useExisting` 以引用注册的提供者，而不是让Nest在令牌背后实例化它。

现在 `JwtAuthGuard` 对Nest来说是作为一个可以覆盖的常规提供者，当创建 `TestingModule` 时：

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideProvider(JwtAuthGuard)
  .useClass(MockAuthGuard)
  .compile();
```

现在所有测试都将在每个请求上使用 `MockAuthGuard`。

#### 测试请求作用域实例

[请求作用域](/fundamentals/injection-scopes) 提供者为每个传入的 **请求** 独特创建。实例在请求处理完成后被垃圾收集。这提出了一个问题，因为我们不能访问为测试请求特别生成的依赖注入子树。

我们知道（基于上述部分），`resolve()` 方法可以用来检索动态实例化的类。此外，如 [这里](https://docs.nestjs.com/fundamentals/module-ref#resolving-scoped-providers) 所述，我们知道可以传递一个唯一的上下文标识符来控制DI容器子树的生命周期。我们如何在测试上下文中利用这一点？

策略是事先生成一个上下文标识符，并强制Nest使用这个特定的ID为所有传入请求创建子树。通过这种方式，我们将能够检索为测试请求创建的实例。

要实现这一点，请使用 `jest.spyOn()` 对 `ContextIdFactory` 进行操作：

```typescript
const contextId = ContextIdFactory.create();
jest
  .spyOn(ContextIdFactory, 'getByRequest')
  .mockImplementation(() => contextId);
```

现在我们可以使用 `contextId` 访问任何后续请求的单个生成的DI容器子树。

```typescript
catsService = await moduleRef.resolve(CatsService, contextId);
```