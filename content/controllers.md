### 控制器

控制器负责处理传入的**请求**并返回**响应**给客户端。

<figure><img class="illustrative-image" src="/assets/Controllers_1.png" /></figure>

控制器的目的是接收应用程序的特定请求。**路由**机制控制哪个控制器接收哪个请求。通常，每个控制器有多个路由，不同的路由可以执行不同的操作。

为了创建一个基本的控制器，我们使用类和**装饰器**。装饰器将类与所需的元数据关联起来，并使Nest能够创建路由图（将请求绑定到相应的控制器）。

> **提示**：为了快速创建带有[验证](https://docs.nestjs.com/techniques/validation)内置的CRUD控制器，您可以使用CLI的[CRUD生成器](https://docs.nestjs.com/recipes/crud-generator#crud-generator)：`nest g resource [name]`。

#### 路由

在以下示例中，我们将使用`@Controller()`装饰器，这是定义基本控制器的**必需**。我们将指定一个可选的路由路径前缀`cats`。在`@Controller()`装饰器中使用路径前缀允许我们轻松地对一组相关路由进行分组，并最小化重复代码。例如，我们可能选择将一组管理与猫实体交互的路由分组到路径`/cats`下。在这种情况下，我们可以在`@Controller()`装饰器中指定路径前缀`cats`，这样我们就不必为文件中的每个路由重复该路径部分。

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

> **提示**：要使用CLI创建控制器，只需执行`$ nest g controller [name]`命令。

`@Get()` HTTP请求方法装饰器在`findAll()`方法之前告诉Nest为特定端点创建一个处理器，用于HTTP请求。端点对应于HTTP请求方法（此例中为GET）和路由路径。什么是路由路径？处理器的路由路径是通过连接（可选的）声明的控制器前缀和方法装饰器中指定的任何路径来确定的。由于我们为每个路由声明了前缀（`cats`），并且没有在装饰器中添加任何路径信息，Nest将映射`GET /cats`请求到此处理器。如上所述，路径包括两个可选的控制器路径前缀**和**任何在请求方法装饰器中声明的路径字符串。例如，路径前缀`cats`与装饰器`@Get('breed')`结合将产生路由映射，用于请求如`GET /cats/breed`。

在我们上面的示例中，当向此端点发出GET请求时，Nest将请求路由到我们用户定义的`findAll()`方法。请注意，我们在这里选择的方法名称是完全任意的。显然，我们必须声明一个方法来绑定路由，但Nest对所选的方法名称没有附加任何意义。

此方法将返回200状态码和相关响应，此例中只是一个字符串。为什么会这样呢？为了解释这一点，我们首先引入Nest使用的两种**不同**的响应操作选项：

<table>
  <tr>
    <td>标准（推荐）</td>
    <td>
      使用此内置方法时，当请求处理器返回JavaScript对象或数组时，它将<strong>自动</strong>被序列化为JSON。当它返回JavaScript原始类型（例如 <code>string</code>，<code>number</code>，<code>boolean</code>）时，Nest将只发送值而不尝试序列化它。这使得响应处理变得简单：只需返回值，Nest会处理其余的事情。
      <br />
      <br /> 此外，响应的<strong>状态码</strong>默认总是200，除了POST请求使用201。我们可以通过在处理器级别添加<code>@HttpCode(...)</code>装饰器轻松更改此行为（见<a href='controllers#status-code'>状态码</a>）。
    </td>
  </tr>
  <tr>
    <td>库特定</td>
    <td>
      我们可以使用库特定的（例如，Express）<a href="https://expressjs.com/en/api.html#res" rel="nofollow" target="_blank">响应对象</a>，通过在方法处理器签名中使用<code>@Res()</code>装饰器注入此对象（例如，<code>findAll(@Res() response)</code>）。通过这种方法，您可以使用该对象暴露的原生响应处理方法。例如，使用Express时，您可以使用像<code>response.status(200).send()</code>这样的代码构建响应。
    </td>
  </tr>
</table>

> **警告**：Nest检测到处理器使用`@Res()`或`@Next()`时，表明您已选择库特定选项。如果同时使用这两种方法，标准方法将**自动禁用**此单一路由，并且不再按预期工作。要同时使用这两种方法（例如，通过注入响应对象仅设置cookie/headers但仍将其余工作留给框架），您必须在`@Res({{ '{' }} passthrough: true {{ '}' }})`装饰器中将`passthrough`选项设置为`true`。

<app-banner-devtools></app-banner-devtools>

#### 请求对象

处理器经常需要访问客户端**请求**详细信息。Nest提供了对底层平台（默认为Express）的[请求对象](https://expressjs.com/en/api.html#req)的访问。我们可以通过在处理器签名中添加`@Req()`装饰器来指示Nest注入请求对象。

```typescript
@@filename(cats.controller)
import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Bind, Get, Req } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  @Bind(Req())
  findAll(request) {
    return 'This action returns all cats';
  }
}
```

> **提示**：为了利用`express`类型（如上例中的`request: Request`参数），安装`@types/express`包。

请求对象代表HTTP请求，具有请求查询字符串、参数、HTTP头和正文（了解更多[here](https://expressjs.com/en/api.html#req)）的属性。在大多数情况下，不需要手动获取这些属性。相反，我们可以使用专用的装饰器，如`@Body()`或`@Query()`，它们都是现成可用的。以下是提供的装饰器和它们代表的纯平台特定对象的列表。

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td>
    </tr>
    <tr>
      <td><code>@Response(), @Res()</code><span class="table-code-asterisk">*</span></td>
      <td><code>res</code></td>
    </tr>
    <tr>
      <td><code>@Next()</code></td>
      <td><code>next</code></td>
    </tr>
    <tr>
      <td><code>@Session()</code></td>
      <td><code>req.session</code></td>
    </tr>
    <tr>
      <td><code>@Param(key?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[key]</code></td>
    </tr>
    <tr>
      <td><code>@Body(key?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[key]</code></td>
    </tr>
    <tr>
      <td><code>@Query(key?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[key]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(name?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[name]</code></td>
    </tr>
    <tr>
      <td><code>@Ip()</code></td>
      <td><code>req.ip</code></td>
    </tr>
    <tr>
      <td><code>@HostParam()</code></td>
      <td><code>req.hosts</code></td>
    </tr>
  </tbody>
</table>

*对于跨底层HTTP平台（例如，Express和Fastify）的类型兼容性，Nest提供了`@Res()`和`@Response()`装饰器。`@Res()`只是`@Response()`的别名。两者都直接暴露底层原生平台`response`对象接口。使用它们时，您还应该导入底层库的类型（例如，`@types/express`）以充分利用。请注意，当您在方法处理器中注入`@Res()`或`@Response()`时，您将Nest置于**库特定模式**，您需要负责管理响应。这样做时，您必须通过在`response`对象上发出某种类型的响应（例如，`res.json(...)`或`res.send(...)`），否则HTTP服务器将挂起。

> **提示**：要了解如何创建自己的自定义装饰器，请访问[本](/custom-decorators)章节。

#### 资源

之前，我们定义了一个获取猫资源的端点（**GET**路由）。我们通常还想提供一个创建新记录的端点。为此，让我们创建**POST**处理器：

```typescript
@@filename(cats.controller)
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create() {
    return 'This action adds a new cat';
  }

  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

就是这么简单。Nest为所有标准HTTP方法提供了装饰器：`@Get()`，`@Post()`，`@Put()`，`@Delete()`，`@Patch()`，`@Options()`和`@Head()`。此外，`@All()`定义了一个处理所有方法的端点。

#### 路由通配符

也支持基于模式的路由。例如，星号用作通配符，将匹配任何字符组合。

```typescript
@Get('ab*cd')
findAll() {
  return 'This route uses a wildcard';
}
```

`'ab*cd'`路由路径将匹配`abcd`，`ab_cd`，`abecd`等。字符`?`，`+`，`*`和`()`可以使用在路由路径中，并且是它们的正则表达式对应物的子集。连字符（`-`）和点（`.`）由基于字符串的路径字面解释。

> **警告**：中间的通配符仅由express支持。

#### 状态码

如上所述，响应**状态码**默认总是**200**，除了POST请求为**201**。我们可以通过在处理器级别添加`@HttpCode(...)`装饰器轻松更改此行为。

```typescript
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

> **提示**：从`@nestjs/common`包导入`HttpCode`。

通常，您的状态码不是静态的，而是取决于各种因素。在这种情况下，您可以使用库特定的**响应**（通过`@Res()`注入）对象（或者，在发生错误的情况下，抛出异常）。

#### 头部

要指定自定义响应头，您可以使用`@Header()`装饰器或库特定的响应对象（直接调用`res.header()`）。

```typescript
@Post()
@Header('Cache-Control', 'no-store')
create() {
  return 'This action adds a new cat';
}
```

> **提示**：从`@nestjs/common`包导入`Header`。

#### 重定向

要将响应重定向到特定URL，您可以使用`@Redirect()`装饰器或库特定的响应对象（直接调用`res.redirect()`）。

`@Redirect()`接受两个参数，`url`和`statusCode`，两者都是可选的。如果省略，默认值`statusCode`是`302`（`Found`）。

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

> **提示**：有时您可能想要动态确定HTTP状态码或重定向URL。通过返回遵循`HttpRedirectResponse`接口（来自`@nestjs/common`）的对象来实现这一点。

返回值将覆盖传递给`@Redirect()`装饰器的任何参数。例如：

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

#### 路由参数

静态路径的路由在需要接受请求中作为部分动态数据时（例如，`GET /cats/1`获取id为`1`的猫）将不起作用。为了定义带有参数的路由，我们可以在路由路径中添加路由参数**标记**以捕获请求URL中该位置的动态值。在`@Get()`装饰器示例中声明的路由参数通过这种方式可以访问，使用`@Param()`装饰器，应添加到方法签名中。

> **提示**：带参数的路由应在任何静态路径之后声明。这可以防止参数化路径截获旨在用于静态路径的流量。

```typescript
@@filename()
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
@@switch
@Get(':id')
@Bind(Param())
findOne(params) {
  console.log(params.id);
  return `This action returns a #${id} cat`;
}
```

`@Param()`用于装饰方法参数（上述示例中的`params`），并使**路由**参数作为该装饰方法参数内部的属性可用。如代码所示，我们可以通过引用`params.id`访问`id`参数。您还可以将特定的参数标记传递给装饰器，然后直接按名称在方法体内引用路由参数。

> **提示**：从`@nestjs/common`包导入`Param`。

```typescript
@@filename()
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
@@switch
@Get(':id')
@Bind(Param('id'))
findOne(id) {
  return `This action returns a #${id} cat`;
}
```

#### 子域路由

`@Controller`装饰器可以采用`host`选项，要求传入请求的HTTP主机与某个特定值匹配。

```typescript
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

> **警告**：由于**Fastify**不支持嵌套路由器，因此在使用子域路由时，应使用（默认）Express适配器。

类似于路由`path`，`hosts`选项可以使用标记来捕获主机名中该位置的动态值。在`@Controller()`装饰器示例中声明的主机参数以这种方式声明，可以使用`@HostParam()`装饰器访问，应添加到方法签名中。

```typescript
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}
```

#### 范围

对于来自不同编程语言背景的人来说，了解到在Nest中，几乎所有内容都在传入请求之间共享，这可能是出乎意料的。我们有一个数据库的连接池，具有全局状态的单例服务等。请记住，Node.js不遵循请求/响应多线程无状态模型，每个请求都由单独的线程处理。因此，在我们的应用程序中使用单例实例是完全**安全**的。

然而，有一些边缘情况，请求基础的控制器生命周期可能是期望的行为，例如GraphQL应用程序中的每请求缓存，请求跟踪或多租户。了解如何控制范围[here](/fundamentals/injection-scopes)。

#### 异步性

我们喜欢现代JavaScript，我们知道数据提取大多是**异步**的。这就是为什么Nest支持并且与`async`函数很好地合作。

> **提示**：了解更多关于`async / await`特性[here](https://kamilmysliwiec.com/typescript-2-1-introduction-async-await)

每个异步函数都必须返回一个`Promise`。这意味着您可以返回一个Nest能够自行解决的延迟值。让我们看一个这样的例子：

```typescript
@@filename(cats.controller)
@Get()
async findAll(): Promise<any[]> {
  return [];
}
@@switch
@Get()
async findAll() {
  return [];
}
```

上述代码是完全有效的。此外，Nest路由处理器甚至更强大，能够返回RxJS[可观察流](https://rxjs-dev.firebaseapp.com/guide/observable)。Nest将自动订阅源并取最后一次发射的值（一旦流完成）。

```typescript
@@filename(cats.controller)
@Get()
findAll(): Observable<any[]> {
  return of([]);
}
@@switch
@Get()
findAll() {
  return of([]);
}
```

上述两种方法都可以工作，您可以使用适合您需求的方法。

#### 请求有效载荷

我们之前的POST路由处理器示例没有接受任何客户端参数。让我们通过添加`@Body()`装饰器来解决这个问题。

但首先（如果您使用TypeScript），我们需要确定**DTO**（数据传输对象）模式。DTO是一个定义如何通过网络发送数据的对象。我们可以使用**TypeScript**接口或简单的类来确定DTO模式。有趣的是，我们建议这里使用**类**。为什么？类是JavaScript ES6标准的一部分，因此在编译后的JavaScript中它们被保留为真实实体。另一方面，由于TypeScript接口在转录过程中被移除，Nest无法在运行时引用它们。这很重要，因为像**Pipes**这样的功能在它们能够在运行时访问变量的元类型时可以提供额外的可能性。

让我们创建`CreateCatDto`类：

```typescript
@@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

它只有三个基本属性。之后我们可以在`CatsController`中使用新创建的DTO：

```typescript
@@filename(cats.controller)
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
@@switch
@Post()
@Bind(Body())
async create(createCatDto) {
  return 'This action adds a new cat';
}
```

> **提示**：我们的`ValidationPipe`可以过滤出不应由方法处理器接收的属性。在这种情况下，我们可以白名单可接受的属性，任何未包含在白名单中的属性都自动从结果对象中剥离。在`CreateCatDto`示例中，我们的白名单是`name`，`age`和`breed`属性。了解更多[here](https://docs.nestjs.com/techniques/validation#stripping-properties)。

#### 错误处理

关于错误处理（即，处理异常）有一个单独的章节[here](/exception-filters)。

#### 完整资源示例

以下示例使用多个可用的装饰器创建一个基本控制器。此控制器公开了几个方法来访问和操作内部数据。

```typescript
@@filename(cats.controller)
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
@@switch
import { Controller, Get, Query, Post, Body, Put, Param, Delete, Bind } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Body())
  create(createCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  @Bind(Query())
  findAll(query) {
    console.log(query);
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  @Bind(Param('id'))
  findOne(id) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  @Bind(Param('id'), Body())
  update(id, updateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  @Bind(Param('id'))
  remove(id) {
    return `This action removes a #${id} cat`;
  }
}
```

> **提示**：Nest CLI提供了一个生成器（schematic），它自动生成**所有样板代码**，以帮助我们避免做所有这些工作，并使开发人员体验更简单。了解更多关于这个特性[here](/recipes/crud-generator)。

#### 启动和运行

有了上述完全定义的控制器，Nest仍然不知道`CatsController`存在，因此不会创建这个类的实例。

控制器总是属于一个模块，这就是我们在`@Module()`装饰器中包含`controllers`数组的原因。由于我们还没有定义任何其他模块，除了根`AppModule`，我们将使用它来引入`CatsController`：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';

@Module({
  controllers: [CatsController],
})
export class AppModule {}
```

我们使用`@Module()`装饰器将元数据附加到模块类，Nest现在可以轻松地反射哪些控制器需要被挂载。

#### 库特定方法

到目前为止，我们已经讨论了Nest标准方式的操作响应。第二种操作响应的方法是使用库特定的[响应对象](https://expressjs.com/en/api.html#res)。为了注入特定的响应对象，我们需要使用`@Res()`装饰器。为了显示差异，让我们将`CatsController`重写如下：

```typescript
@@filename()
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
    res.status(HttpStatus.OK).json([]);
  }
}
@@switch
import { Controller, Get, Post, Bind, Res, Body, HttpStatus } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Res(), Body())
  create(res, createCatDto) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  @Bind(Res())
  findAll(res) {
    res.status(HttpStatus.OK).json([]);
  }
}
```

虽然这种方法有效，并且实际上在某些方面提供了更多的灵活性（通过提供对响应对象的完全控制，例如头部操作，库特定功能等），但它应该谨慎使用。总的来说，这种方法不太清晰，并且有一些缺点。主要缺点是您的代码变得平台依赖性（因为底层库可能在响应对象上有不同的API），并且更难测试（您将不得不模拟响应对象等）。

此外，在上述示例中，您失去了与依赖于Nest标准响应处理的Nest功能的兼容性，例如拦截器和`@HttpCode()` / `@Header()`装饰器。要解决这个问题，您可以将`passthrough`选项设置为`true`，如下所示：

```typescript
@@filename()
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
@@switch
@Get()
@Bind(Res({ passthrough: true }))
findAll(res) {
  res.status(HttpStatus.OK);
  return [];
}
```

现在，您可以与原生响应对象交互（例如，根据某些条件设置cookie或头部），但其余工作留给框架。