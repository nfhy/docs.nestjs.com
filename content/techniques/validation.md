### 验证

最佳实践是验证发送到任何Web应用程序的数据的正确性。为了自动验证传入请求，Nest 提供了几个内置的管道：

- `ValidationPipe`
- `ParseIntPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`

`ValidationPipe` 利用功能强大的 [class-validator](https://github.com/typestack/class-validator) 包及其声明式验证装饰器。`ValidationPipe` 提供了一种方便的方法来强制执行所有传入客户端负载的验证规则，其中特定规则是用简单的注释在每个模块的本地类/DTO声明中声明的。

#### 概览

在 [Pipes](/pipes) 章节中，我们通过构建简单的管道并将它们绑定到控制器、方法或全局应用程序来演示这个过程是如何工作的。请务必查看该章节，以便最好地理解本章的主题。这里，我们将专注于 `ValidationPipe` 的各种**现实世界**用例，并展示如何使用它的一些高级定制功能。

#### 使用内置的 ValidationPipe

要开始使用它，我们首先安装所需的依赖项。

```bash
$ npm i --save class-validator class-transformer
```

> info **提示** `ValidationPipe` 是从 `@nestjs/common` 包中导出的。

由于这个管道使用了 [`class-validator`](https://github.com/typestack/class-validator) 和 [`class-transformer`](https://github.com/typestack/class-transformer) 库，因此有许多选项可供选择。您可以通过传递给管道的配置对象来配置这些设置。以下是内置选项：

```typescript
export interface ValidationPipeOptions extends ValidatorOptions {
  transform?: boolean;
  disableErrorMessages?: boolean;
  exceptionFactory?: (errors: ValidationError[]) => any;
}
```

除此之外，所有 `class-validator` 选项（从 `ValidatorOptions` 接口继承）都可用：

<table>
  <tr>
    <th>选项</th>
    <th>类型</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>enableDebugMessages</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，则当某些事情不正确时，验证器将向控制台打印额外的警告消息。</td>
  </tr>
  <tr>
    <td><code>skipUndefinedProperties</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，则验证器将跳过验证对象中所有未定义的属性的验证。</td>
  </tr>
  <tr>
    <td><code>skipNullProperties</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，则验证器将跳过验证对象中所有 null 的属性的验证。</td>
  </tr>
  <tr>
    <td><code>skipMissingProperties</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，则验证器将跳过验证对象中所有 null 或未定义的属性的验证。</td>
  </tr>
  <tr>
    <td><code>whitelist</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证器将从返回的验证对象中剥离任何未使用任何验证装饰器的属性。</td>
  </tr>
  <tr>
    <td><code>forbidNonWhitelisted</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证器不会剥离非白名单属性，而是抛出异常。</td>
  </tr>
  <tr>
    <td><code>forbidUnknownValues</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，尝试验证未知对象将立即失败。</td>
  </tr>
  <tr>
    <td><code>disableErrorMessages</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证错误将不会返回给客户端。</td>
  </tr>
  <tr>
    <td><code>errorHttpStatusCode</code></td>
    <td><code>number</code></td>
    <td>此设置允许您指定在发生错误时使用哪种异常类型。默认情况下，它抛出 <code>BadRequestException</code>。</td>
  </tr>
  <tr>
    <td><code>exceptionFactory</code></td>
    <td><code>Function</code></td>
    <td>接收一个验证错误的数组，并返回要抛出的异常对象。</td>
  </tr>
  <tr>
    <td><code>groups</code></td>
    <td><code>string[]</code></td>
    <td>在对象验证期间使用的组。</td>
  </tr>
  <tr>
    <td><code>always</code></td>
    <td><code>boolean</code></td>
    <td>设置装饰器的 <code>always</code> 选项的默认值。默认值可以在装饰器选项中被覆盖。</td>
  </tr>
  <tr>
    <td><code>strictGroups</code></td>
    <td><code>boolean</code></td>
    <td>如果 <code>groups</code> 未给出或为空，则忽略至少有一个组的装饰器。</td>
  </tr>
  <tr>
    <td><code>dismissDefaultMessages</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证将不使用默认消息。如果未明确设置，则错误消息始终为 <code>undefined</code>。</td>
  </tr>
  <tr>
    <td><code>validationError.target</code></td>
    <td><code>boolean</code></td>
    <td>指示是否应在 <code>ValidationError</code> 中暴露目标。</td>
  </tr>
  <tr>
    <td><code>validationError.value</code></td>
    <td><code>boolean</code></td>
    <td>指示是否应在 <code>ValidationError</code> 中暴露验证值。</td>
  </tr>
  <tr>
    <td><code>stopAtFirstError</code></td>
    <td><code>boolean</code></td>
    <td>当设置为 true 时，给定属性的验证在遇到第一个错误后将停止。默认为 false。</td>
  </tr>
</table>

> info **注意** 在其 [仓库](https://github.com/typestack/class-validator) 中查找更多关于 `class-validator` 包的信息。

#### 自动验证

我们将从在应用程序级别绑定 `ValidationPipe` 开始，以确保所有端点都能防止接收到错误的数据。

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

为了测试我们的管道，让我们创建一个基本的端点。

```typescript
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return 'This action adds a new user';
}
```

> info **提示** 由于 TypeScript 不存储有关 **泛型或接口** 的元数据，当您在 DTO 中使用它们时，`ValidationPipe` 可能无法正确验证传入的数据。因此，考虑在 DTO 中使用具体类。

> info **提示** 导入您的 DTO 时，您不能使用仅类型导入，因为那将在运行时被擦除，即记住要 `import { CreateUserDto }` 而不是 `import type { CreateUserDto }`。

现在我们可以在 `CreateUserDto` 中添加一些验证规则。我们使用 `class-validator` 包提供的装饰器来这样做，详细描述在 [这里](https://github.com/typestack/class-validator#validation-decorators)。通过这种方式，任何使用 `CreateUserDto` 的路由都将自动强制执行这些验证规则。

```typescript
import { IsEmail, IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsNotEmpty()
  password: string;
}
```

有了这些规则，如果请求以请求正文中的无效 `email` 属性击中我们的端点，应用程序将自动响应 `400 Bad Request` 代码，以及以下响应正文：

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be an email"]
}
```

除了验证请求正文之外，`ValidationPipe` 还可以与其它请求对象属性一起使用。想象一下，我们想要接受端点路径中的 `:id`。为确保这个请求参数只接受数字，我们可以使用以下结构：

```typescript
@Get(':id')
findOne(@Param() params: FindOneParams) {
  return 'This action returns a user';
}
```

`FindOneParams`，像 DTO 一样，只是一个使用 `class-validator` 定义验证规则的类。它看起来像这样：

```typescript
import { IsNumberString } from 'class-validator';

export class FindOneParams {
  @IsNumberString()
  id: number;
}
```

#### 禁用详细错误

错误消息可以帮助解释请求中的错误。然而，一些生产环境更喜欢禁用详细错误。通过向 `ValidationPipe` 传递一个选项对象来实现这一点：

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    disableErrorMessages: true,
  }),
);
```

结果，详细错误消息将不会在响应正文中显示。

#### 剥离属性

我们的 `ValidationPipe` 还可以过滤出方法处理程序不应接收的属性。在这种情况下，我们可以**白名单**可接受的属性，任何不在白名单中的属性都会自动从结果对象中剥离。例如，如果我们的处理程序期望 `email` 和 `password` 属性，但请求还包括一个 `age` 属性，这个属性可以自动从结果 DTO 中移除。要启用这种行为，请将 `whitelist` 设置为 `true`。

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
  }),
);
```

当设置为 true 时，这将自动移除非白名单属性（那些在验证类中没有任何装饰器的属性）。

或者，当存在非白名单属性时，您可以停止处理请求，并返回错误响应给用户。要启用此功能，请将 `forbidNonWhitelisted` 选项属性设置为 `true`，并结合设置 `whitelist` 为 `true`。

#### 转换负载对象

通过网络传输的负载是纯 JavaScript 对象。`ValidationPipe` 可以自动将负载转换为根据其 DTO 类型进行类型化的