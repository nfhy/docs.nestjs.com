### 会话
**HTTP会话**提供了一种在多个请求之间存储用户信息的方法，这对于[MVC](/techniques/mvc)应用程序特别有用。

#### 在Express中使用（默认）
首先安装[所需的包](https://github.com/expressjs/session)（以及TypeScript用户的类型定义）：
```shell
$ npm i express-session
$ npm i -D @types/express-session
```
安装完成后，将`express-session`中间件作为全局中间件应用（例如，在您的`main.ts`文件中）。
```typescript
import * as session from 'express-session';
// 在您的初始化文件中的某个位置
app.use(
  session({
    secret: 'my-secret',
    resave: false,
    saveUninitialized: false,
  }),
);
```
> 警告 **注意** 默认的服务器端会话存储并不适合生产环境。它在大多数情况下会泄露内存，不能扩展到单个进程之外，并且仅用于调试和开发。更多信息请阅读[官方仓库](https://github.com/expressjs/session)。

`secret`用于签名会话ID cookie。这可以是用于单个密钥的字符串，或者用于多个密钥的数组。如果提供了密钥数组，只有第一个元素将用于签名会话ID cookie，而所有元素在请求中验证签名时都会被考虑。密钥本身不应该容易被人解析，最好是一组随机字符。

启用`resave`选项会强制将未在请求中修改的会话保存回会话存储。默认值是`true`，但使用默认值已被弃用，因为默认值将来会改变。

同样，启用`saveUninitialized`选项会强制将“未初始化”的会话保存到存储中。当会话是新的但未被修改时，会话就是未初始化的。选择`false`对于实现登录会话、减少服务器存储使用或遵守要求在设置cookie之前获得许可的法律很有用。选择`false`还可以帮助解决客户端在没有会话的情况下并行发出多个请求的竞态条件（来源：[链接](https://github.com/expressjs/session#saveuninitialized)）。

您可以向`session`中间件传递其他几个选项，更多信息请阅读[API文档](https://github.com/expressjs/session#options)。

> 提示 **提示** 请注意`secure: true`是一个推荐的选项。但是，它需要一个启用了https的网站，即需要HTTPS才能设置安全的cookie。如果设置了secure，并且您通过HTTP访问您的网站，cookie将不会被设置。如果您的node.js在代理后面，并且使用了`secure: true`，您需要在express中设置`\"trust proxy\"`。

有了这些设置，您现在可以在路由处理程序中设置和读取会话值，如下所示：
```typescript
@Get()
findAll(@Req() request: Request) {
  request.session.visits = request.session.visits ? request.session.visits + 1 : 1;
}
```
> 提示 **提示** `@Req()`装饰器从`@nestjs/common`导入，而`Request`从`express`包导入。

或者，您可以使用`@Session()`装饰器从请求中提取会话对象，如下所示：
```typescript
@Get()
findAll(@Session() session: Record<string, any>) {
  session.visits = session.visits ? session.visits + 1 : 1;
}
```
> 提示 **提示** `@Session()`装饰器从`@nestjs/common`包导入。

#### 在Fastify中使用
首先安装所需的包：
```shell
$ npm i @fastify/secure-session
```
安装完成后，注册`fastify-secure-session`插件：
```typescript
import secureSession from '@fastify/secure-session';

// 在您的初始化文件中的某个位置
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
await app.register(secureSession, {
  secret: 'averylogphrasebiggerthanthirtytwochars',
  salt: 'mq9hDxBVDbspDR6n',
});
```
> 提示 **提示** 您也可以预生成一个密钥（[查看说明](https://github.com/fastify/fastify-secure-session)）或使用[密钥轮换](https://github.com/fastify/fastify-secure-session#using-keys-with-key-rotation)。

更多可用选项的信息，请阅读[官方仓库](https://github.com/fastify/fastify-secure-session)。

有了这些设置，您现在可以在路由处理程序中设置和读取会话值，如下所示：
```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  const visits = request.session.get('visits');
  request.session.set('visits', visits ? visits + 1 : 1);
}
```
或者，您可以使用`@Session()`装饰器从请求中提取会话对象，如下所示：
```typescript
@Get()
findAll(@Session() session: secureSession.Session) {
  const visits = session.get('visits');
  session.set('visits', visits ? visits + 1 : 1);
}
```
> 提示 **提示** `@Session()`装饰器从`@nestjs/common`导入，而`secureSession.Session`从`@fastify/secure-session`包导入（导入语句：`import * as secureSession from '@fastify/secure-session'`）。