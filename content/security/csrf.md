### CSRF保护

跨站请求伪造（CSRF或XSRF）是一种攻击类型，攻击者通过这种方式发送来自受信任用户的**未经授权**命令到一个Web应用程序。为了帮助防止这种攻击，可以使用[csrf-csrf](https://github.com/Psifi-Solutions/csrf-csrf)包。

#### 在Express中使用（默认）

首先，安装所需的包：

```bash
$ npm i csrf-csrf
```

> 警告 **警告** 如[csrf-csrf文档](https://github.com/Psifi-Solutions/csrf-csrf?tab=readme-ov-file#getting-started)中所指出的，这个中间件需要在之前初始化会话中间件或`cookie-parser`。请参考文档以获取更多详细信息。

安装完成后，将`csrf-csrf`中间件注册为全局中间件。

```typescript
import { doubleCsrf } from 'csrf-csrf';
// ...
// 在你的初始化文件中的某个位置
const {
  invalidCsrfTokenError, // 如果你计划创建自己的中间件，这个仅提供方便。
  generateToken, // 在你的路由中使用这个来生成并提供CSRF哈希，以及一个token cookie和token。
  validateRequest, // 如果你计划制作自己的中间件，这也是一个方便的工具。
  doubleCsrfProtection, // 这是默认的CSRF保护中间件。
} = doubleCsrf(doubleCsrfOptions);
app.use(doubleCsrfProtection);
```

#### 在Fastify中使用

首先，安装所需的包：

```bash
$ npm i --save @fastify/csrf-protection
```

安装完成后，按照以下方式注册`@fastify/csrf-protection`插件：

```typescript
import fastifyCsrf from '@fastify/csrf-protection';
// ...
// 在你的初始化文件中，在注册一些存储插件后的某个位置
await app.register(fastifyCsrf);
```

> 警告 **警告** 如`@fastify/csrf-protection`文档[这里](https://github.com/fastify/csrf-protection#usage)所解释的，这个插件需要先初始化一个存储插件。请查看该文档以获取进一步的指导。