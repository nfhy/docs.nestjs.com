### 管道

管道是一个用 `@Injectable()` 装饰器注解的类，它实现了 `PipeTransform` 接口。

<figure>
  <img class="illustrative-image" src="/assets/Pipe_1.png" />
</figure>

管道有两种典型的用例：

- **转换**：将输入数据转换为所需的形式（例如，从字符串转换为整数）
- **验证**：评估输入数据，如果有效，则原样传递；否则，抛出异常

在这两种情况下，管道都是在由<a href="controllers#route-parameters">控制器路由处理器</a>处理的`参数`上操作。Nest在方法被调用前插入一个管道，该管道接收预定给方法的参数并对它们进行操作。任何转换或验证操作都在此时进行，之后路由处理器将使用（可能已经转换的）参数被调用。

Nest自带了一些开箱即用的内置管道。你也可以构建你自己的自定义管道。在本章中，我们将介绍内置管道，并展示如何将它们绑定到路由处理器。然后，我们将检查几个自定义构建的管道，以展示如何从头开始构建一个管道。

> **提示**：管道在异常区域内部运行。这意味着当管道抛出异常时，它会被异常层（全局异常过滤器和任何应用于当前上下文的[异常过滤器](/exception-filters)）处理。鉴于此，当管道中抛出异常时，很明显随后不会执行任何控制器方法。这为你提供了一种最佳实践技术，用于在系统边界处验证来自外部源的数据。

#### 内置管道

Nest自带了九个管道：

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`

它们都是从 `@nestjs/common` 包中导出的。

让我们快速看一下如何使用 `ParseIntPipe`。这是一个**转换**用例的例子，其中管道确保方法处理器参数被转换为JavaScript整数（如果转换失败则抛出异常）。在本章稍后，我们将展示一个简单的自定义 `ParseIntPipe` 实现。下面的示例技术也适用于其他内置转换管道（`ParseBoolPipe`、`ParseFloatPipe`、`ParseEnumPipe`、`ParseArrayPipe` 和 `ParseUUIDPipe`，我们将在本章中将这些称为 `Parse*` 管道）。

#### 绑定管道

要使用管道，我们需要将管道类的实例绑定到适当的上下文。在我们的 `ParseIntPipe` 示例中，我们希望将管道与特定的路由处理器方法关联，并确保在方法被调用之前运行它。我们使用以下结构来实现这一点，我们将称之为在方法参数级别绑定管道：

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

这确保了以下两种情况之一为真：我们收到的 `findOne()` 方法中的参数是一个数字（正如我们在调用 `this.catsService.findOne()` 中所期望的），或者在路由处理器被调用之前抛出异常。

例如，假设路由被调用如下：

```bash
GET localhost:3000/abc
```

Nest将抛出如下异常：

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

异常将阻止 `findOne()` 方法体的执行。

在上面的例子中，我们传递了一个类（`ParseIntPipe`），而不是一个实例，将实例化的责任留给了框架，并启用了依赖注入。与管道和守卫一样，我们也可以通过传递一个就地实例来实现。传递一个就地实例在我们要通过传递选项来自定义内置管道的行为时很有用：

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

绑定其他转换管道（所有的 **Parse\*** 管道）的工作方式类似。这些管道都在验证路由参数、查询字符串参数和请求体值的上下文中工作。

例如，使用查询字符串参数：

```typescript
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

这里有一个使用 `ParseUUIDPipe` 解析字符串参数并验证它是否是UUID的例子。

```typescript
@filename()
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
@switch
@Get(':uuid')
@Bind(Param('uuid', new ParseUUIDPipe()))
async findOne(uuid) {
  return this.catsService.findOne(uuid);
}
```

> **提示**：当使用 `ParseUUIDPipe()` 时，你正在解析版本3、4或5的UUID，如果你只需要特定版本的UUID，你可以在管道选项中传递一个版本。

以上我们看到了绑定各种 `Parse*` 系列内置管道的例子。绑定验证管道有点不同；我们将在下一节讨论。

> **提示**：另见[验证技术](/techniques/validation)以获取验证管道的广泛示例。

#### 自定义管道

如上所述，你可以构建你自己的自定义管道。虽然Nest提供了强大的内置 `ParseIntPipe` 和 `ValidationPipe`，让我们从头开始构建每个管道的简单自定义版本，以了解自定义管道是如何构建的。

我们从一个简单的 `ValidationPipe` 开始。最初，它将接收一个输入值并立即返回相同的值，表现得像一个恒等函数。

```typescript
@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class ValidationPipe {
  transform(value, metadata) {
    return value;
  }
}
```

> **提示**：`PipeTransform<T, R>` 是一个泛型接口，任何管道都必须实现此接口。泛型接口使用 `T` 表示输入 `value` 的类型，`R` 表示 `transform()` 方法的返回类型。

每个管道都必须实现 `transform()` 方法以满足 `PipeTransform` 接口契约。这个方法有两个参数：

- `value`
- `metadata`

`value` 参数是当前处理的方法参数（在被路由处理方法接收之前），`metadata` 是当前处理的方法参数的元数据。元数据对象具有以下属性：

```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

这些属性描述了当前处理的参数。

<table>
  <tr>
    <td>
      <code>type</code>
    </td>
    <td>表示参数是体<code>@Body()</code>、查询<code>@Query()</code>、参数<code>@Param()</code>还是自定义参数（更多信息请<a routerLink="/custom-decorators">点击这里</a>）。</td>
  </tr>
  <tr>
    <td>
      <code>metatype</code>
    </td>
    <td>提供参数的元类型，例如<code>String</code>。注意：如果你在路由处理器方法签名中省略类型声明，或者使用纯JavaScript，该值将是<code>undefined</code>。</td>
  </tr>
  <tr>
    <td>
      <code>data</code>
    </td>
    <td>传递给装饰器的字符串，例如<code>@Body('string')</code>。如果装饰器的括号为空，则为<code>undefined</code>。</td>
  </tr>
</table>

> **警告**：TypeScript接口在转译过程中会消失。因此，如果方法参数的类型被声明为接口而不是类，`metatype` 的值将是 `Object`。

#### 基于模式的验证

让我们使我们的验证管道更有用一些。让我们更仔细地看看 `CatsController` 的 `create()` 方法，我们可能希望确保在尝试运行我们的服务方法之前，post体对象是有效的。

```typescript
@filename()
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
async create(@Body() createCatDto) {
  this.catsService.create(createCatDto);
}
```

让我们关注 `createCatDto` 体参数。它的类型是 `CreateCatDto`：

```typescript
@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

我们希望确保任何传入创建方法的请求都包含一个有效的体。因此我们必须验证 `createCatDto` 对象的三个成员。我们可以在路由处理器方法内这样做，但这样做并不理想，因为它会违反**单一责任原则**（SRP）。

另一种方法是创建一个**验证器类**并将任务委托给它。这样做的缺点是我们将不得不记住在每个方法的开始调用这个验证器。

创建验证中间件怎么样？这可能是可行的，但不幸的是，不可能创建可以跨整个应用程序的所有上下文使用的**通用中间件**。这是因为中间件不了解**执行上下文**，包括将被调用的处理程序和它的任何参数。

这当然是管道设计的确切用例。所以让我们继续完善我们的验证管道。

#### 基于对象模式的验证

在干净、[DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)的方式中，有几种方法可用于对象验证。一种常见的方法是使用**基于模式**的验证。让我们尝试这种方法。

[Zod](https://zod.dev/) 库允许你以直接的方式创建模式，具有可读的API。让我们构建一个使用基于Zod模式的验证管道。

首先，安装所需的包：

```bash
$ npm install --save zod
```

在下面的代码示例中，我们创建了一个简单的类，它接受一个模式作为 `constructor` 参数。然后我们应用 `schema.parse()` 方法，该方法验证我们的传入参数是否符合提供的模式。

如前所述，**验证管道**要么原样返回值，要么抛出异常。

在下一节中，你将看到我们如何为给定的控制器方法提供适当的模式，使用 `@UsePipes()` 装饰器。这样做使我们的验证管道可以在不同上下文中重用，正如我们所计划的。

```typescript
@filename()
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema  } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
@@switch
import { BadRequestException } from '@nestjs/common';

export class ZodValidationPipe {
  constructor(private schema) {}

  transform(value, metadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
```

#### 绑定验证管道

早些时候，我们看到了如何绑定转换管道（如 `ParseIntPipe` 和其他 `Parse*` 管道）。

绑定验证管道也非常简单。

在这种情况下，我们希望在方法调用级别绑定管道。在我们当前的例子中，我们需要做以下事情来使用 `ZodValidationPipe`：

1. 创建 `ZodValidationPipe` 的实例
2. 在管道的类构造函数中传递上下文特定的Zod模式
3. 将管道绑定到方法

Zod模式示例：

```typescript
import { z } from 'zod';

export const createCatSchema = z
  .object({
    name: z.string(),
    age: z.number(),
    breed: z.string(),
  })
  .required();

export type CreateCatDto = z.infer<typeof createCatSchema>;
```

我们使用 `@UsePipes()` 装饰器如下所示：

```typescript
@filename(cats.controller)
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Bind(Body())
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> **提示**：`@UsePipes()` 装饰器从 `@nestjs/common` 包中导入。

> **警告**：`zod` 库要求在 `tsconfig.json` 文件中启用 `strictNullChecks` 配置。

#### 类验证器

> **警告**：本节中的技术需要TypeScript，如果你的应用程序是用纯JavaScript编写的，则不可用。

让我们看看我们的验证技术的另一种实现。

Nest与[class-validator](https://github.com/typestack/class-validator)库配合得很好。这个强大的库允许你使用基于装饰器的验证。基于装饰器的验证非常强大，特别是当与Nest的**管道**功能结合使用时，因为我们有访问到处理属性的 `metatype`。在我们开始之前，我们需要安装所需的包：

```bash
$ npm i --save class-validator class-transformer
```

一旦这些安装完成，我们可以在 `CreateCatDto` 类上添加一些装饰器。在这里我们看到了这种方法的一个显著优势：`CreateCatDto` 类仍然是我们的Post体对象的唯一真相来源（而不是不得不创建一个单独的验证类）。

```typescript
@filename(create-cat.dto)
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

> **提示**：更多关于class-validator装饰器的信息[点击这里](https://github.com/typestack/class-validator#usage)。

现在我们可以创建一个使用这些注释的 `ValidationPipe` 类。

```typescript
@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

> **提示**：作为提醒，你不必自己构建通用验证管道，因为Nest提供了开箱即用的 `ValidationPipe`。内置的 `ValidationPipe` 提供的选项比我们在本章中构建的示例更多，本示例保持了基础，以说明自定义构建管道的机制。你可以在[这里](/techniques/validation)找到更多详细信息和许多示例。

> **注意**：我们在上面使用了[class-transformer](https://github.com/typestack/class-transformer)库，它是由 **class-validator** 库的同一作者制作的，因此它们配合得非常好。

让我们通过这段代码。首先，请注意 `transform()` 方法被标记为 `async`。这是可能的，因为Nest支持同步和**异步**管道。我们使这个方法 `async`，因为一些class-validator验证[可以是异步的](https://github.com/typestack/class-validator#custom-validation-classes)（使用Promises）。

接下来请注意，我们使用解构来提取元类型字段（从 `ArgumentMetadata` 中提取此成员）到我们的 `metatype` 参数。这只是获取完整的 `ArgumentMetadata` 然后有一个额外的语句来分配元类型变量的简写。

接下来，请注意帮助函数 `toValidate()`。它负责在当前处理的参数是原生JavaScript类型时绕过验证步骤（这些不能附加验证装饰器，因此没有理由让它们通过验证步骤）。

接下来，我们使用class-transformer函数 `plainToInstance()` 将我们的纯JavaScript参数对象转换为类型化对象，以便我们可以应用验证。我们必须这样做的原因是，传入的post体对象在从网络请求中反序列化时**没有任何类型信息**（这是底层平台，如Express的工作方式）。class-validator需要使用我们为DTO定义的验证装饰器，所以我们需要执行这种转换，以将传入的体视为适当装饰的对象，而不仅仅是一个普通的对象。

最后，如前所述，由于这是一个**验证管道**，它要么原样返回值，要么抛出异常。

最后一步是绑定 `ValidationPipe`。管道可以是参数范围的、方法范围的、控制器范围的或全局范围的。早些时候，使用我们的基于Zod的验证管道，我们看到了在方法级别绑定管道的例子。在下面的例子中，我们将管道实例绑定到路由处理器 `@Body()` 装饰器，以便我们的管道被调用以验证post体。

```typescript
@filename(cats.controller)
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

参数范围的管道在验证逻辑仅涉及一个指定参数时很有用。

#### 全局范围管道

由于 `ValidationPipe` 被创建为尽可能通用，我们可以通过将其设置为**全局范围**的管道来实现其全部效用，以便它被应用于整个应用程序中的每个路由处理器。

```typescript
@filename(main)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> **注意**：在<a href="faq/hybrid-application">混合应用程序</a>的情况下，`useGlobalPipes()` 方法不会为网关和微服务设置管道。对于“标准”（非混合）微服务应用程序，`useGlobalPipes()` 确实会全局安装管道。

全局管道在整个应用程序中使用，适用于每个控制器和每个路由处理器。

请注意，在依赖注入方面，从模块外部注册的全局管道（如上例中的 `useGlobalPipes()`）不能注入依赖项，因为绑定已经在任何模块的上下文之外完成。为了解决这个问题，你可以直接从任何模块设置全局管道：

```typescript
@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

> **提示**：当使用这种方法为管道执行依赖注入时，请注意，无论在哪个模块中使用这种构造，管道实际上是全局的。应该在哪里执行这个操作？选择定义管道的模块（上面的示例中的 `ValidationPipe`）。另外，`useClass` 不是处理自定义提供程序注册的唯一方式。了解更多[点击这里](/fundamentals/custom-providers)。

#### 内置的ValidationPipe

作为提醒，你不必自己构建通用验证管道，因为Nest提供了开箱即用的 `ValidationPipe`。内置的 `ValidationPipe` 提供的选项比我们在本章中构建的示例更多，本示例保持了基础，以说明自定义构建管道的机制。你可以在[这里](/techniques/validation)找到更多详细信息和许多示例。

#### 转换用例

验证不是自定义管道的唯一用例。在本章开始时，我们提到管道也可以**转换**输入数据为所需的格式。这是可能的，因为从 `transform` 函数返回的值完全覆盖了参数的先前值。

这在什么时候有用？考虑有时从客户端传递的数据需要在路由处理器方法处理之前进行一些更改 - 例如将字符串转换为整数。此外，一些所需的数据字段可能缺失，我们希望应用默认值。**转换管道**可以通过在客户端请求和请求处理器之间插入一个处理函数来执行这些功能。

这里有一个简单的 `ParseIntPipe`，它负责将字符串解析为整数值。（如上所述，Nest有一个更复杂的内置 `ParseIntPipe`；我们将其作为一个自定义转换管道的简单示例包含在内）。

```typescript
@filename(parse-int.pipe)
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
@@switch
import { Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe {
  transform(value, metadata) {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

然后我们可以如下所示将这个管道绑定到选定的参数：

```typescript
@filename()
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
@@switch
@Get(':id')
@Bind(Param('id', new ParseIntPipe()))
async findOne(id) {
  return this.catsService.findOne(id);
}
```

另一个有用的转换案例将是使用请求中提供的id从数据库中选择一个**现有的用户**实体：

```typescript
@filename()
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
@@switch
@Get(':id')
@Bind(Param('id', UserByIdPipe))
findOne(userEntity) {
  return userEntity;
}
```

我们将这个管道的实现留给读者，但请注意，像所有其他转换管道一样，它接收一个输入值（一个 `id`）并返回一个输出值（一个 `UserEntity` 对象）。这可以使你的代码更加声明性和[DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)，通过将样板代码从你的处理器抽象出来，放入一个通用的管道中。

#### 提供默认值

`Parse*` 管道期望参数的值是已定义的。它们在接收到 `null` 或 `undefined` 值时会抛出异常。为了允许端点处理缺失的查询字符串参数值，我们必须提供一个默认值，以便在 `Parse*` 管道操作这些值之前注入。`DefaultValuePipe` 就是为此目的服务的。简单地在 `@Query()` 装饰器中实例化一个 `DefaultValuePipe`，然后是相关的 `Parse*` 管道，如下所示：

```typescript
@filename()
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```