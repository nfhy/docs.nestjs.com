### 版本控制

> 信息 **提示** 这一章节仅与基于HTTP的应用程序相关。

版本控制允许您在同一应用程序中运行**不同版本**的控制器或单个路由。应用程序经常发生变化，需要进行破坏性更改的同时，通常还需要支持应用程序的上一版本。

支持的版本控制类型有4种：

| 类型 | 描述 |
| --- | --- |
| [URI版本控制](techniques/versioning#uri-versioning-type) | 版本将通过请求的URI传递（默认） |
| [Header版本控制](techniques/versioning#header-versioning-type) | 自定义请求头将指定版本 |
| [媒体类型版本控制](techniques/versioning#media-type-versioning-type) | 请求的`Accept`头将指定版本 |
| [自定义版本控制](techniques/versioning#custom-versioning-type) | 可以使用请求的任何方面来指定版本。提供了一个自定义函数来提取版本 |

#### URI版本控制类型

URI版本控制使用请求URI中传递的版本，例如`https://example.com/v1/route`和`https://example.com/v2/route`。

> 注意 与URI版本控制一起，版本将自动添加到URI中全局路径前缀（如果存在）之后，以及任何控制器或路由路径之前。

要为您的应用程序启用URI版本控制，请执行以下操作：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
// 或 "app.enableVersioning()"
app.enableVersioning({
  type: VersioningType.URI,
});
await app.listen(process.env.PORT ?? 3000);
```

> 注意 URI中的版本默认会自动以`v`为前缀，但是可以通过设置`prefix`键为您希望的前缀或`false`（如果您希望禁用它）来配置前缀值。

> 提示 `VersioningType`枚举可用于`type`属性，并且从`@nestjs/common`包中导入。

#### Header版本控制类型

Header版本控制使用自定义的、用户指定的请求头来指定版本，其中头部的值将是用于请求的版本。

Header版本控制的示例HTTP请求：

要为您的应用程序启用**Header版本控制**，请执行以下操作：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'Custom-Header',
});
await app.listen(process.env.PORT ?? 3000);
```

`header`属性应该是包含请求版本号的头部名称。

> 提示 `VersioningType`枚举可用于`type`属性，并且从`@nestjs/common`包中导入。

#### 媒体类型版本控制类型

媒体类型版本控制使用请求的`Accept`头来指定版本。

在`Accept`头中，版本将通过分号`;`与媒体类型分隔。它应该包含一个键值对，表示用于请求的版本，例如`Accept: application/json;v=2`。键被视为前缀，用于确定版本将配置为包含键和分隔符。

要为您的应用程序启用**媒体类型版本控制**，请执行以下操作：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key: 'v=',
});
await app.listen(process.env.PORT ?? 3000);
```

`key`属性应该是包含版本的键值对的键和分隔符。对于示例`Accept: application/json;v=2`，`key`属性将设置为`v=`。

> 提示 `VersioningType`枚举可用于`type`属性，并且从`@nestjs/common`包中导入。

#### 自定义版本控制类型

自定义版本控制使用请求的任何方面来指定版本（或版本）。传入的请求使用`extractor`函数进行分析，该函数返回一个字符串或字符串数组。

如果请求者提供了多个版本，`extractor`函数可以返回一个字符串数组，按从最高/最高版本到最低/最低版本的顺序排序。版本将按从最高到最低的顺序与路由匹配。

如果`extractor`返回空字符串或数组，则没有路由匹配，并返回404。

例如，如果传入请求指定它支持版本`1`、`2`和`3`，则`extractor`**必须**返回`[3, 2, 1]`。这确保首先选择最高可能的路由版本。

如果提取了版本`[3, 2, 1]`，但只存在版本`2`和`1`的路由，则选择匹配版本`2`的路由（自动忽略版本`3`）。

> 注意 基于从`extractor`返回的数组选择最高匹配版本**不可靠**地与Express适配器一起工作，由于设计限制。Express中的单个版本（要么是字符串，要么是1个元素的数组）可以正常工作。Fastify正确支持最高匹配版本选择和单个版本选择。

要为您的应用程序启用**自定义版本控制**，请创建一个`extractor`函数并将其传递到您的应用程序中，如下所示：

```typescript
@@filename(main)
// 示例提取器从自定义头部提取版本列表，并将其转换为排序数组。
// 此示例使用Fastify，但Express请求可以以类似的方式处理。
const extractor = (request: FastifyRequest): string | string[] =>
  [request.headers['custom-versioning-field'] ?? '']
    .flatMap(v => v.split(','))
    .filter(v => !!v)
    .sort()
    .reverse()

const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.CUSTOM,
  extractor,
});
await app.listen(process.env.PORT ?? 3000);
```

#### 使用

版本控制允许您对控制器、单个路由进行版本控制，并还提供了一种方式，使某些资源可以退出版本控制。无论您的应用程序使用哪种版本控制类型，版本控制的使用都是相同的。

> 注意 如果为应用程序启用了版本控制，但控制器或路由没有指定版本，则对该控制器/路由的任何请求都将返回`404`响应状态。类似地，如果收到包含没有相应控制器或路由版本的请求，也将返回`404`响应状态。

#### 控制器版本

可以对控制器应用版本，为控制器内的所有路由设置版本。

要为控制器添加版本，请执行以下操作：

```typescript
@@filename(cats.controller)
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1';
  }
}
@@switch
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1';
  }
}
```

#### 路由版本

可以对单个路由应用版本。此版本将覆盖任何其他可能影响路由的版本，例如控制器版本。

要为单个路由添加版本，请执行以下操作：

```typescript
@@filename(cats.controller)
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1(): string {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2(): string {
    return 'This action returns all cats for version 2';
  }
}
@@switch
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1() {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2() {
    return 'This action returns all cats for version 2';
  }
}
```

#### 多个版本

可以对控制器或路由应用多个版本。要使用多个版本，您将版本设置为数组。

要添加多个版本，请执行以下操作：

```typescript
@@filename(cats.controller)
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1 or 2';
  }
}
@@switch
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1 or 2';
  }
}
```

#### 版本“中性”

有些控制器或路由可能不关心版本，并且无论版本如何，都将具有相同的功能。为了适应这一点，可以将版本设置为`VERSION_NEUTRAL`符号。

传入的请求将被映射到`VERSION_NEUTRAL`控制器或路由，无论请求中发送的版本如何，此外，如果请求不包含版本。

> 注意 对于URI版本控制，`VERSION_NEUTRAL`资源将不会在URI中出现版本。

要添加版本中性控制器或路由，请执行以下操作：

```typescript
@@filename(cats.controller)
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats regardless of version';
  }
}
@@switch
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats regardless of version';
  }
}
```

#### 全局默认版本

如果您不想为每个控制器/单个路由提供版本，或者如果您想要为没有指定版本的每个控制器/路由设置特定版本作为默认版本，您可以如下设置`defaultVersion`：

```typescript
@@filename(main)
app.enableVersioning({
  // ...
  defaultVersion: '1'
  // 或
  defaultVersion: ['1', '2']
  // 或
  defaultVersion: VERSION_NEUTRAL
});
```

#### 中间件版本控制

[中间件](https://docs.nestjs.com/middleware)也可以使用版本控制元数据来为特定路由的版本配置中间件。为此，请将版本号作为参数之一提供给`MiddlewareConsumer.forRoutes()`方法：

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET, version: '2' });
  }
}
```

上述代码中，`LoggerMiddleware`将仅应用于`/cats`端点的版本'2'。

> 注意 中间件适用于本节中描述的任何版本控制类型：`URI`、`Header`、`Media Type`或`Custom`。