### Suites（原名 Automock）

Suites 是一个有主见且灵活的测试元框架，旨在增强后端系统的软件测试体验。通过将各种测试工具整合到一个统一的框架中，Suites 简化了可靠测试的创建，帮助确保开发高质量的软件。

> **提示**：`Suites` 是第三方包，不是由 NestJS 核心团队维护的。请将任何与该库相关的问题报告给[相应的仓库](https://github.com/suites-dev/suites)。

#### 引言

控制反转（IoC）是 NestJS 框架的一个基本原则，它使得架构模块化、可测试。虽然 NestJS 提供了内置工具来创建测试模块，但 Suites 提供了一种替代方法，强调测试隔离的单元或小团体单元。Suites 使用依赖项的虚拟容器，其中自动生成模拟对象，消除了手动将每个提供者替换为 IoC（或 DI）容器中的模拟对象的需要。这种方法可以单独使用，也可以与 NestJS 的 `Test.createTestingModule` 方法一起使用，根据您的需求提供更多的灵活性进行单元测试。

#### 安装

要将 Suites 与 NestJS 一起使用，请安装必要的包：

```bash
$ npm i -D @suites/unit @suites/di.nestjs @suites/doubles.jest
```

> **提示**：`Suites` 也支持 Vitest 和 Sinon 作为测试替身，分别是 `@suites/doubles.vitest` 和 `@suites/doubles.sinon`。

#### 示例和模块设置

考虑一个包含 `CatsApiService`、`CatsDAL`、`HttpClient` 和 `Logger` 的 `CatsService` 模块设置。这将是本食谱示例的基础：

```typescript
@@filename(cats.module)
import { HttpModule } from '@nestjs/axios';
import { PrismaModule } from '../prisma.module';

@Module({
  imports: [HttpModule.register({ baseUrl: 'https://api.cats.com/' }), PrismaModule],
  providers: [CatsService, CatsApiService, CatsDAL, Logger],
  exports: [CatsService],
})
export class CatsModule {}
```

`HttpModule` 和 `PrismaModule` 都向宿主模块导出提供者。

让我们从隔离测试 `CatsHttpService` 开始。这项服务负责从 API 获取猫的数据并记录操作。

```typescript
@@filename(cats-http.service)
@Injectable()
export class CatsHttpService {
  constructor(private httpClient: HttpClient, private logger: Logger) {}

  async fetchCats(): Promise<Cat[]> {
    this.logger.log('Fetching cats from the API');
    const response = await this.httpClient.get('/cats');
    return response.data;
  }
}
```

我们想要隔离 `CatsHttpService` 并模拟其依赖项 `HttpClient` 和 `Logger`。Suites 允许我们使用 `TestBed` 的 `.solitary()` 方法轻松地做到这一点。

```typescript
@@filename(cats-http.service.spec)
import { TestBed, Mocked } from '@suites/unit';

describe('Cats Http Service Unit Test', () => {
  let catsHttpService: CatsHttpService;
  let httpClient: Mocked<HttpClient>;
  let logger: Mocked<Logger>;

  beforeAll(async () => {
    // 隔离 CatsHttpService 并模拟 HttpClient 和 Logger
    const { unit, unitRef } = await TestBed.solitary(CatsHttpService).compile();

    catsHttpService = unit;
    httpClient = unitRef.get(HttpClient);
    logger = unitRef.get(Logger);
  });

  it('should fetch cats from the API and log the operation', async () => {
    const catsFixtures: Cat[] = [{ id: 1, name: 'Catty' }, { id: 2, name: 'Mitzy' }];
    httpClient.get.mockResolvedValue({ data: catsFixtures });

    const cats = await catsHttpService.fetchCats();

    expect(logger.log).toHaveBeenCalledWith('Fetching cats from the API');
    expect(httpClient.get).toHaveBeenCalledWith('/cats');
    expect(cats).toEqual<Cat[]>(catsFixtures);
  });
});
```

在上面的例子中，Suites 自动模拟了 `CatsHttpService` 的依赖项，使用 `TestBed.solitary()`。这使得设置更加容易，因为您不必手动模拟每个依赖项。

- 自动模拟依赖项：Suites 为被测试单元的所有依赖项生成模拟对象。
- 模拟的空行为：最初，这些模拟没有任何预定义的行为。您需要根据测试需要指定它们的行为。
- `unit` 和 `unitRef` 属性：
  - `unit` 指的是被测试类的实际实例，包括其模拟的依赖项。
  - `unitRef` 是一个引用，允许您访问模拟的依赖项。

#### 使用 `TestingModule` 测试 `CatsApiService`

对于 `CatsApiService`，我们希望确保 `HttpModule` 在 `CatsModule` 宿主模块中正确导入和配置。这包括验证 `Axios` 的基础 URL（和其他配置）是否设置正确。

在这种情况下，我们不会使用 Suites；相反，我们将使用 Nest 的 `TestingModule` 来测试 `HttpModule` 的实际配置。我们将使用 `nock` 来模拟 HTTP 请求，而不是在这种情况下模拟 `HttpClient`。

```typescript
@@filename(cats-api.service)
import { HttpClient } from '@nestjs/axios';

@Injectable()
export class CatsApiService {
  constructor(private httpClient: HttpClient) {}

  async getCatById(id: number): Promise<Cat> {
    const response = await this.httpClient.get(`/cats/${id}`);
    return response.data;
  }
}
```

我们需要测试 `CatsApiService`，使用真实的、未模拟的 `HttpClient` 以确保 DI 和 `Axios`（http）的配置正确。这涉及到导入 `CatsModule` 并使用 `nock` 进行 HTTP 请求模拟。

```typescript
@@filename(cats-api.service.integration.test)
import { Test } from '@nestjs/testing';
import * as nock from 'nock';

describe('Cats Api Service Integration Test', () => {
  let catsApiService: CatsApiService;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    }).compile();

    catsApiService = moduleRef.get(CatsApiService);
  });

  afterEach(() => {
    nock.cleanAll();
  });

  it('should fetch cat by id using real HttpClient', async () => {
    const catFixture: Cat = { id: 1, name: 'Catty' };

    nock('https://api.cats.com') // 使这个 URL 与 HttpModule 注册中的 URL 相同
      .get('/cats/1')
      .reply(200, catFixture);

    const cat = await catsApiService.getCatById(1);
    expect(cat).toEqual<Cat>(catFixture);
  });
});
```

#### 社交测试示例

接下来，让我们测试 `CatsService`，它依赖于 `CatsApiService` 和 `CatsDAL`。我们将模拟 `CatsApiService` 并暴露 `CatsDAL`。

```typescript
@@filename(cats.dal)
import { PrismaClient } from '@prisma/client';

@Injectable()
export class CatsDAL {
  constructor(private prisma: PrismaClient) {}

  async saveCat(cat: Cat): Promise<Cat> {
    return this.prisma.cat.create({ data: cat });
  }
}
```

接下来，我们有 `CatsService`，它依赖于 `CatsApiService` 和 `CatsDAL`：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    private catsApiService: CatsApiService,
    private catsDAL: CatsDAL
  ) {}

  async getAndSaveCat(id: number): Promise<Cat> {
    const cat = await this.catsApiService.getCatById(id);
    return this.catsDAL.saveCat(cat);
  }
}
```

现在，让我们使用 Suites 的社交测试来测试 `CatsService`：

```typescript
@@filename(cats.service.spec)
import { TestBed, Mocked } from '@suites/unit';
import { PrismaClient } from '@prisma/client';

describe('Cats Service Sociable Unit Test', () => {
  let catsService: CatsService;
  let prisma: Mocked<PrismaClient>;
  let catsApiService: Mocked<CatsApiService>;

  beforeAll(async () => {
    // 社交测试设置，暴露 CatsDAL 并模拟 CatsApiService
    const { unit, unitRef } = await TestBed.sociable(CatsService)
      .expose(CatsDAL)
      .mock(CatsApiService)
      .final({ getCatById: async () => ({ id: 1, name: 'Catty' })})
      .compile();

    catsService = unit;
    prisma = unitRef.get(PrismaClient);
  });

  it('should get cat by id and save it', async () => {
    const catFixture: Cat = { id: 1, name: 'Catty' };
    prisma.cat.create.mockResolvedValue(catFixture);

    const savedCat = await catsService.getAndSaveCat(1);

    expect(prisma.cat.create).toHaveBeenCalledWith({ data: catFixture });
    expect(savedCat).toEqual(catFixture);
  });
});
```

在这个例子中，我们使用 `.sociable()` 方法来设置测试环境。我们利用 `.expose()` 方法允许与 `CatsDAL` 的真实交互，同时使用 `.mock()` 方法模拟 `CatsApiService`。`.final()` 方法为 `CatsApiService` 建立了固定的行为，确保跨测试的一致结果。

这种方法强调测试 `CatsService` 与 `CatsDAL` 的真实交互，涉及处理 `Prisma`。在这种情况下，Suites 将使用 `CatsDAL` 作为是，只有其依赖项，如 `Prisma`，才会被模拟。

需要注意的是，这种方法**仅用于验证行为**，与加载整个测试模块不同。社交测试对于确认单元在隔离其直接依赖项时的行为和交互非常有价值。

#### 集成测试和数据库

对于 `CatsDAL`，可以测试真实的数据库，如 SQLite 或 PostgreSQL（例如，使用 Docker Compose）。然而，在这个例子中，我们将模拟 `Prisma` 并专注于社交测试。模拟 `Prisma` 的原因是为了避免 I/O 操作，专注于 `CatsService` 的行为。也就是说，您也可以进行带有真实 I/O 操作和实时数据库的测试。

#### 社交单元测试、集成测试和模拟

- 社交单元测试：这些专注于测试单元之间的交互和行为，同时模拟它们更深层的依赖项。在这个例子中，我们模拟 `Prisma` 并暴露 `CatsDAL`。

- 集成测试：涉及真实的 I/O 操作和完全配置的依赖注入（DI）设置。测试带有 `HttpModule` 和 `nock` 的 `CatsApiService` 被认为是集成测试，因为它验证了 `HttpClient` 的真实配置和交互。在这种情况下，我们将使用 Nest 的 `TestingModule` 来加载实际的模块配置。

**在使用模拟时要谨慎。** 确保测试 I/O 操作和 DI 配置（特别是当涉及 HTTP 或数据库交互时）。在集成测试中验证这些组件后，您可以自信地模拟它们进行社交单元测试，专注于行为和交互。Suites 社交测试旨在验证单元在隔离其直接依赖项时的行为，而集成测试确保整个系统的配置和 I/O 操作功能正确。

#### 测试 IoC 容器注册

验证您的 DI 容器正确配置以防止运行时错误至关重要。这包括确保所有提供者、服务和模块都正确注册和注入。测试 DI 容器配置有助于及早捕捉配置错误，防止可能仅在运行时出现的问题。

为了确认 IoC 容器设置正确，让我们创建一个集成测试，加载实际的模块配置并验证所有提供者都已注册和正确注入。

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { CatsModule } from './cats.module';
import { CatsService } from './cats.service';

describe('Cats Module Integration Test', () => {
  let moduleRef: TestingModule;

  beforeAll(async () => {
    moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    }).compile();
  });

  it('should resolve exported providers from the ioc container', () => {
    const catsService = moduleRef.get(CatsService);
    expect(catsService).toBeDefined();
  });
});
```

#### 单独、社交、集成和 E2E 测试之间的比较

#### 单独单元测试

- **焦点**：在完全隔离的情况下测试单个单元（类）。
- **用例**：测试 `CatsHttpService`。
- **工具**：Suites 的 `TestBed.solitary()` 方法。
- **示例**：模拟 `HttpClient` 并测试 `CatsHttpService`。

#### 社交单元测试

- **焦点**：在模拟更深层依赖项的同时验证单元之间的交互。
- **用例**：测试带有模拟 `CatsApiService` 的 `CatsService` 并暴露 `CatsDAL`。
- **工具**：Suites 的 `TestBed.sociable()` 方法。
- **示例**：模拟 `Prisma` 并测试 `CatsService`。

#### 集成测试

- **焦点**：涉及真实的 I/O 操作和完全配置的模块（IoC 容器）。
- **用例**：测试带有 `HttpModule` 和 `nock` 的 `CatsApiService`。
- **工具**：Nest 的 `TestingModule`。
- **示例**：测试 `HttpClient` 的真实配置和交互。

#### E2E 测试

- **焦点**：覆盖类和模块在更聚合的层面上的交互。
- **用例**：从最终用户的角度测试系统的完整行为。
- **工具**：Nest 的 `TestingModule`，`supertest`。
- **示例**：使用 `supertest` 模拟 HTTP 请求测试 `CatsModule`。

有关设置和运行 E2E 测试的更多详细信息，请参考 [NestJS 官方测试指南](https://docs.nestjs.com/fundamentals/testing#end-to-end-testing)。