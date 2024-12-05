### Cookies

一个**HTTP cookie**是用户浏览器存储的一小块数据。Cookies被设计为一种可靠的机制，用于网站记住有状态的信息。当用户再次访问网站时，cookie会自动随请求发送。

#### 在Express中的使用（默认）

首先安装[所需的包](https://github.com/expressjs/cookie-parser)（以及TypeScript用户的类型定义）：

```shell
$ npm i cookie-parser
$ npm i -D @types/cookie-parser
```

安装完成后，将`cookie-parser`中间件作为全局中间件应用（例如，在你的`main.ts`文件中）。

```typescript
import * as cookieParser from 'cookie-parser';
// 在你的初始化文件中的某个地方
app.use(cookieParser());
```

你可以向`cookieParser`中间件传递几个选项：

- `secret` 一个字符串或数组，用于签名cookie。这是可选的，如果没有指定，则不会解析签名的cookie。如果提供了字符串，这个字符串将被用作密钥。如果提供了数组，将尝试按顺序使用每个密钥来取消签名cookie。
- `options` 一个对象，作为第二个选项传递给`cookie.parse`。更多信息请查看[cookie](https://www.npmjs.org/package/cookie)。

中间件将解析请求上的`Cookie`头，并暴露cookie数据作为属性`req.cookies`，如果提供了密钥，则作为属性`req.signedCookies`。这些属性是cookie名称到cookie值的键值对。

当提供了密钥时，该模块将取消签名并验证任何签名的cookie值，并将这些键值对从`req.cookies`移动到`req.signedCookies`。一个签名的cookie是一个值以`s:`为前缀的cookie。签名验证失败的签名cookie将有值`false`而不是被篡改的值。

有了这个设置，你现在可以在路由处理器中读取cookie，如下所示：

```typescript
@Get()
findAll(@Req() request: Request) {
  console.log(request.cookies); // 或 "request.cookies['cookieKey']"
  // 或 console.log(request.signedCookies);
}
```

> 信息提示 **Hint** `@Req()`装饰器是从`@nestjs/common`导入的，而`Request`是从`express`包导入的。

要将cookie附加到传出的响应中，使用`Response#cookie()`方法：

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: Response) {
  response.cookie('key', 'value')
}
```

> 警告 **Warning** 如果你想将响应处理逻辑留给框架，记得像上面所示那样将`passthrough`选项设置为`true`。更多信息[在这里](/controllers#library-specific-approach)。

> 信息提示 **Hint** `@Res()`装饰器是从`@nestjs/common`导入的，而`Response`是从`express`包导入的。

#### 在Fastify中的使用

首先安装所需的包：

```shell
$ npm i @fastify/cookie
```

安装完成后，注册`@fastify/cookie`插件：

```typescript
import fastifyCookie from '@fastify/cookie';

// 在你的初始化文件中的某个地方
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
await app.register(fastifyCookie, {
  secret: 'my-secret', // 用于cookie签名
});
```

有了这个设置，你现在可以在路由处理器中读取cookie，如下所示：

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  console.log(request.cookies); // 或 "request.cookies['cookieKey']"
}
```

> 信息提示 **Hint** `@Req()`装饰器是从`@nestjs/common`导入的，而`FastifyRequest`是从`fastify`包导入的。

要将cookie附加到传出的响应中，使用`FastifyReply#setCookie()`方法：

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: FastifyReply) {
  response.setCookie('key', 'value')
}
```

要了解更多关于`FastifyReply#setCookie()`方法的信息，请查看这个[页面](https://github.com/fastify/fastify-cookie#sending)。

> 警告 **Warning** 如果你想将响应处理逻辑留给框架，记得像上面所示那样将`passthrough`选项设置为`true`。更多信息[在这里](/controllers#library-specific-approach)。

> 信息提示 **Hint** `@Res()`装饰器是从`@nestjs/common`导入的，而`FastifyReply`是从`fastify`包导入的。

#### 创建自定义装饰器（跨平台）

为了提供一种方便的、声明式的方式来访问传入的cookie，我们可以创建一个[自定义装饰器](/custom-decorators)。

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const Cookies = createParamDecorator((data: string, ctx: ExecutionContext) => {
  const request = ctx.switchToHttp().getRequest();
  return data ? request.cookies?.[data] : request.cookies;
});
```

`@Cookies()`装饰器将从`req.cookies`对象中提取所有cookie，或一个命名的cookie，并将该值填充到装饰的参数中。

有了这个设置，我们现在可以在路由处理器签名中使用装饰器，如下所示：

```typescript
@Get()
findAll(@Cookies('name') name: string) {}
```