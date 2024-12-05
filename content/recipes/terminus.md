### 健康检查（Terminus）

Terminus集成为您提供了**就绪/存活**健康检查。在复杂的后端设置中，健康检查至关重要。简而言之，健康检查在Web开发领域通常包括一个特殊地址，例如`https://my-website.com/health/readiness`。您的基础设施的一个服务或组件（例如，[Kubernetes](https://kubernetes.io/)）会持续检查这个地址。根据对健康检查地址发出的`GET`请求返回的HTTP状态码，服务将在收到“不健康”响应时采取行动。

由于“健康”或“不健康”的定义因您提供的服务类型而异，**Terminus**集成为您提供了一组**健康指标**。

例如，如果您的Web服务器使用MongoDB存储数据，MongoDB是否仍在运行是至关重要的信息。在这种情况下，您可以利用`MongooseHealthIndicator`。如果配置正确——稍后会详细介绍——您的健康检查地址将根据MongoDB是否运行返回健康或不健康的HTTP状态码。

#### 开始使用

要开始使用`@nestjs/terminus`，我们需要安装所需的依赖项。

```bash
$ npm install --save @nestjs/terminus
```

#### 设置健康检查

健康检查代表**健康指标**的摘要。健康指标执行对服务的检查，无论其处于健康还是不健康状态。如果所有分配的健康指标都在运行，健康检查就是积极的。由于许多应用程序将需要类似的健康指标，[`@nestjs/terminus`](https://github.com/nestjs/terminus)提供了一组预定义的指标，例如：

- `HttpHealthIndicator`
- `TypeOrmHealthIndicator`
- `MongooseHealthIndicator`
- `SequelizeHealthIndicator`
- `MikroOrmHealthIndicator`
- `PrismaHealthIndicator`
- `MicroserviceHealthIndicator`
- `GRPCHealthIndicator`
- `MemoryHealthIndicator`
- `DiskHealthIndicator`

要开始我们的第一个健康检查，让我们创建`HealthModule`并将`TerminusModule`导入到它的导入数组中。

> 信息 **提示** 使用[Nest CLI](cli/overview)创建模块，只需执行`$ nest g module health`命令。

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';

@Module({
  imports: [TerminusModule]
})
export class HealthModule {}
```

我们的健康检查可以使用[控制器](/controllers)执行，使用[Nest CLI](cli/overview)可以轻松设置。

```bash
$ nest g controller health
```

> 信息 **信息** 强烈建议在您的应用程序中启用关闭钩子。Terminus集成在启用此生命周期事件时会使用它。更多关于关闭钩子的信息[在这里](fundamentals/lifecycle-events#application-shutdown)。

#### HTTP健康检查

一旦我们安装了`@nestjs/terminus`，导入了`TerminusModule`并创建了一个新的控制器，我们就可以创建一个健康检查了。

`HTTPHealthIndicator`需要`@nestjs/axios`包，因此请确保已安装：

```bash
$ npm i --save @nestjs/axios axios
```

现在我们可以设置我们的`HealthController`：

```typescript
@@filename(health.controller)
import { Controller, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator, HealthCheck } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
    ]);
  }
}
@@switch
import { Controller, Dependencies, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator, HealthCheck } from '@nestjs/terminus';

@Controller('health')
@Dependencies(HealthCheckService, HttpHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private http,
  ) { }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
    ])
  }
}
```

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
@@switch
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
```

现在我们的健康检查将向`https://docs.nestjs.com`地址发送一个_GET_请求。如果我们从该地址获得健康响应，我们的路由`http://localhost:3000/health`将以200状态码返回以下对象。

```json
{
  "status": "ok",
  "info": {
    "nestjs-docs": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "nestjs-docs": {
      "status": "up"
    }
  }
}
```

这个响应对象的接口可以从`@nestjs/terminus`包中使用`HealthCheckResult`接口访问。

|           |                                                                                                                                                                                             |                                      |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| `status`  | 如果任何健康指标失败，状态将是`'error'`。如果NestJS应用程序正在关闭但仍接受HTTP请求，健康检查将具有`'shutting_down'`状态。 | `'error' \| 'ok' \| 'shutting_down'` |
| `info`    | 包含每个健康指标的信息对象，其状态为`'up'`，或者换句话说“健康”。                                                                              | `object`                             |
| `error`   | 包含每个健康指标的信息对象，其状态为`'down'`，或者换句话说“不健康”。                                                                          | `object`                             |
| `details` | 包含每个健康指标的所有信息对象                                                                                                                                  | `object`                             |

##### 检查特定的HTTP响应代码

在某些情况下，您可能想要检查特定标准并验证响应。例如，假设`https://my-external-service.com`返回响应代码`204`。使用`HttpHealthIndicator.responseCheck`，您可以特别检查该响应代码，并确定所有其他代码为不健康。

如果返回的任何其他响应代码不是`204`，以下示例将是不健康的。第三个参数要求您提供一个函数（同步或异步），该函数返回响应是否被认为是健康的（`true`）或不健康的（`false`）。

```typescript
@@filename(health.controller)
// 在`HealthController`类中

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () =>
      this.http.responseCheck(
        'my-external-service',
        'https://my-external-service.com',
        (res) => res.status === 204,
      ),
  ]);
}
```

#### TypeOrm健康指标

Terminus提供了将数据库检查添加到您的健康检查中的能力。要开始使用此健康指标，请查看[数据库章节](/techniques/sql)并确保您的应用程序中的数据库连接已建立。

> 信息 **提示** 在幕后，`TypeOrmHealthIndicator`简单地执行一个`SELECT 1`-SQL命令，这通常用于验证数据库是否仍然活跃。如果您使用的是Oracle数据库，它使用`SELECT 1 FROM DUAL`。

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, TypeOrmHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private db,
  ) { }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ])
  }
}
```

如果您的数据库可以访问，您现在应该在请求`http://localhost:3000/health`时看到一个`GET`请求的以下JSON结果：

```json
{
  "status": "ok",
  "info": {
    "database": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "database": {
      "status": "up"
    }
  }
}
```

如果您的应用程序使用[多个数据库](techniques/database#multiple-databases)，您需要将每个连接注入到您的`HealthController`中。然后，您可以简单地将连接引用传递给`TypeOrmHealthIndicator`。

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    @InjectConnection('albumsConnection')
    private albumsConnection: Connection,
    @InjectConnection()
    private defaultConnection: Connection,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('albums-database', { connection: this.albumsConnection }),
      () => this.db.pingCheck('database', { connection: this.defaultConnection }),
    ]);
  }
}
```

#### 磁盘健康指标

使用`DiskHealthIndicator`，我们可以检查使用了多少存储空间。要开始，请确保将`DiskHealthIndicator`注入到您的`HealthController`中。以下示例检查路径`/`（或在Windows上您可以使用`C:\\`）的存储使用情况。如果超过总存储空间的50%，则响应将是不健康的健康检查。

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly disk: DiskHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.5 }),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, DiskHealthIndicator)
export class HealthController {
  constructor(health, disk) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.5 }),
    ])
  }
}
```

使用`DiskHealthIndicator.checkStorage`函数，您还可以检查固定数量的空间。以下示例将不健康，如果路径`/my-app/`超过250GB。

```typescript
@@filename(health.controller)
// 在`HealthController`类中

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.disk.checkStorage('storage', {  path: '/', threshold: 250 * 1024 * 1024 * 1024, })
  ]);
}
```

#### 内存健康指标

为确保您的进程不超过某个内存限制，可以使用`MemoryHealthIndicator`。以下示例用于检查进程的堆。

> 信息 **提示** 堆是动态分配内存所在的内存部分（即通过malloc分配的内存）。从堆分配的内存将保持分配状态，直到发生以下情况之一：
> - 内存被_free_d
> - 程序终止

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private memory: MemoryHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, MemoryHealthIndicator)
export class HealthController {
  constructor(health, memory) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ])
  }
}
```

您还可以使用`MemoryHealthIndicator.checkRSS`验证进程的内存RSS。以下示例将不健康，如果您的进程分配了超过150MB。

> 信息 **提示** RSS是驻留集大小，用于显示分配给该进程的内存量以及在RAM中。它不包括被交换出去的内存。只要那些库的页面实际上在内存中，它就包括共享库的内存。它包括所有栈和堆内存。

```typescript
@@filename(health.controller)
// 在`HealthController`类中

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.memory.checkRSS('memory_rss', 150 * 1024 * 1024),
  ]);
}
```

#### 自定义健康指标

在某些情况下，`@nestjs/terminus`提供的预定义健康指标不能满足您的所有健康检查要求。在这种情况下，您可以根据需要设置自定义健康指标。

让我们开始创建一个将代表我们的自定义指标的服务。为了基本了解指标的结构，我们将创建一个示例`DogHealthIndicator`。如果每个`Dog`对象的类型都是`'goodboy'`，则此服务应具有状态`'up'`。如果不满足该条件，则应抛出错误。

```typescript
@@filename(dog.health)
import { Injectable } from '@nestjs/common';
import { HealthIndicator, HealthIndicatorResult, HealthCheckError } from '@nestjs/terminus';

export interface Dog {
  name: string;
  type: string;
}

@Injectable()
export class DogHealthIndicator extends HealthIndicator {
  private dogs: Dog[] = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex', type: 'badboy' },
  ];

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    const badboys = this.dogs.filter(dog => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;
    const result = this.getStatus(key, isHealthy, { badboys: badboys.length });

    if (isHealthy) {
      return result;
    }
    throw new HealthCheckError('Dogcheck failed', result);
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { HealthCheckError } from '@godaddy/terminus';

@Injectable()
export class DogHealthIndicator extends HealthIndicator {
  dogs = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex', type: 'badboy' },
  ];

  async isHealthy(key) {
    const badboys = this.dogs.filter(dog => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;
    const result = this.getStatus(key, isHealthy, { badboys: badboys.length });

    if (isHealthy) {
      return result;
    }
    throw new HealthCheckError('Dogcheck failed', result);
  }
}
```

接下来，我们需要将健康指标注册为提供者。

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { DogHealthIndicator } from './dog.health';

@Module({
  controllers: [HealthController],
  imports: [TerminusModule],
  providers: [DogHealthIndicator]
})
export class HealthModule { }
```

> 信息 **提示** 在现实世界的应用程序中，`DogHealthIndicator`应该在单独的模块中提供，例如`DogModule`，然后由`HealthModule`导入。

最后一步是将现在可用的健康指标添加到所需的健康检查端点。为此，我们回到`HealthController`并将它添加到我们的`check`函数中。

```typescript
@@filename(health.controller)
import { HealthCheckService, HealthCheck } from '@nestjs/terminus';
import { Injectable, Dependencies, Get } from '@nestjs/common';
import { DogHealthIndicator } from './dog.health';

@Injectable()
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private dogHealthIndicator: DogHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.dogHealthIndicator.isHealthy('dog'),
    ])
  }
}
@@switch
import { HealthCheckService, HealthCheck } from '@nestjs/terminus';
import { Injectable, Get } from '@nestjs/common';
import { DogHealthIndicator } from './dog.health';

@Injectable()
@Dependencies(HealthCheckService, DogHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private dogHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.dogHealthIndicator.isHealthy('dog'),
    ])
  }
}
```

#### 日志记录

Terminus仅记录错误消息，例如当健康检查失败时。使用`TerminusModule.forRoot()`方法，您可以更好地控制错误如何被记录，甚至完全接管日志记录本身。

在本节中，我们将向您介绍如何创建自定义日志记录器`TerminusLogger`。这个日志记录器扩展了内置日志记录器。因此，您可以选择要覆盖的日志记录器的哪一部分。

> 信息 **信息** 如果您想了解更多关于NestJS中的自定义日志记录器，[请阅读更多](/techniques/logger#injecting-a-custom-logger)。

```typescript
@@filename(terminus-logger.service)
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class TerminusLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string): void;
  error(message: any, ...optionalParams: any[]): void;
  error(
    message: unknown,
    stack?: unknown,
    context?: unknown,
    ...rest: unknown[]
  ): void {
    // 在这里重写错误消息应该如何记录
  }
}
```

一旦您创建了自定义日志记录器，您只需要简单地将其传递到`TerminusModule.forRoot()`中。

```typescript
@@filename(health.module)
@Module({
imports: [
  TerminusModule.forRoot({
    logger: TerminusLogger,
  }),
],
})
export class HealthModule {}
```

要完全抑制来自Terminus的任何日志消息，包括错误消息，请这样配置Terminus。

```typescript
@@filename(health.module)
@Module({
imports: [
  TerminusModule.forRoot({
    logger: false,
  }),
],
})
export class HealthModule {}
```

Terminus允许您配置健康检查错误应该如何显示在您的日志中。

| 错误日志样式          | 描述                                                                                                                        | 示例                                                              |
|:------------------|:-----------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------|
| `json`  (默认) | 如果发生错误，打印健康检查结果的摘要作为JSON对象                                                     | <figure><img src="/assets/Terminus_Error_Log_Json.png" /></figure>   |
| `pretty`          | 如果发生错误，在格式化框中打印健康检查结果的摘要，并突出显示成功/错误的结果 | <figure><img src="/assets/Terminus_Error_Log_Pretty.png" /></figure> |

您可以使用`errorLogStyle`配置选项更改日志样式，如下所示。

```typescript
@@filename(health.module)
@Module({
  imports: [
    TerminusModule.forRoot({
      errorLogStyle: 'pretty',
    }),
  ]
})
export class HealthModule {}
```

#### 优雅关闭超时

如果您的应用程序需要推迟其关闭过程，Terminus可以为您处理。当与Kubernetes等编排器一起工作时，此设置特别有益。通过将延迟设置得略长于就绪检查间隔，您可以在关闭容器时实现零停机时间。

```typescript
@@filename(health.module)
@Module({
  imports: [
    TerminusModule.forRoot({
      gracefulShutdownTimeoutMs: 1000,
    }),
  ]
})
export class HealthModule {}
```

#### 更多示例

更多工作示例可在[此处](https://github.com/nestjs/terminus/tree/master/sample)找到。