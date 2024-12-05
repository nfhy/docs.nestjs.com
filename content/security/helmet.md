### Helmet

[Helmet](https://github.com/helmetjs/helmet) 可以通过适当设置 HTTP 头来帮助保护您的应用程序免受一些众所周知的网络漏洞的影响。通常情况下，Helmet 只是一组较小的中间件函数，用于设置与安全相关的 HTTP 头（阅读[更多](https://github.com/helmetjs/helmet#how-it-works)）。

> 信息 **提示** 请注意，将 `helmet` 作为全局中间件应用或注册时，必须在其他调用 `app.use()` 或可能调用 `app.use()` 的设置函数之前进行。这是由于底层平台（例如 Express 或 Fastify）的工作方式，其中中间件/路由定义的顺序很重要。如果您在定义路由后使用像 `helmet` 或 `cors` 这样的中间件，那么该中间件将不适用于该路由，它只会适用于在中间件之后定义的路由。

#### 在 Express 中使用（默认）

首先，安装所需的包。

```bash
$ npm i --save helmet
```

安装完成后，将其作为全局中间件应用。

```typescript
import helmet from 'helmet';
// 在您的初始化文件中的某个位置
app.use(helmet());
```

> 警告 **警告** 当使用 `helmet`、`@apollo/server`（4.x）和[Apollo Sandbox](https://docs.nestjs.com/graphql/quick-start#apollo-sandbox)时，可能会遇到[CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)在Apollo Sandbox上的问题。要解决此问题，请按照以下方式配置CSP：

```typescript
app.use(helmet({
  crossOriginEmbedderPolicy: false,
  contentSecurityPolicy: {
    directives: {
      imgSrc: [`'self'`, 'data:', 'apollo-server-landing-page.cdn.apollographql.com'],
      scriptSrc: [`'self'`, `https: 'unsafe-inline'`],
      manifestSrc: [`'self'`, 'apollo-server-landing-page.cdn.apollographql.com'],
      frameSrc: [`'self'`, 'sandbox.embed.apollographql.com'],
    },
  },
}));
```

#### 在 Fastify 中使用

如果您使用的是 `FastifyAdapter`，请安装 [@fastify/helmet](https://github.com/fastify/fastify-helmet) 包：

```bash
$ npm i --save @fastify/helmet
```

[fastify-helmet](https://github.com/fastify/fastify-helmet) 不应该作为中间件使用，而应该作为 [Fastify插件](https://www.fastify.io/docs/latest/Reference/Plugins/) 使用，即通过 `app.register()`：

```typescript
import helmet from '@fastify/helmet'
// 在您的初始化文件中的某个位置
await app.register(helmet)
```

> 警告 **警告** 当使用 `apollo-server-fastify` 和 `@fastify/helmet` 时，可能会遇到[CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)在GraphQL playground上的问题，要解决这种冲突，请按照以下方式配置CSP：

```typescript
await app.register(fastifyHelmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: [`'self'`, 'unpkg.com'],
      styleSrc: [
        `'self'`,
        `'unsafe-inline'`,
        'cdn.jsdelivr.net',
        'fonts.googleapis.com',
        'unpkg.com',
      ],
      fontSrc: [`'self'`, 'fonts.gstatic.com', 'data:'],
      imgSrc: [`'self'`, 'data:', 'cdn.jsdelivr.net'],
      scriptSrc: [
        `'self'`,
        `https: 'unsafe-inline'`,
        `cdn.jsdelivr.net`,
        `'unsafe-eval'`,
      ],
    },
  },
});

// 如果您根本不打算使用CSP，可以使用以下方式：
await app.register(fastifyHelmet, {
  contentSecurityPolicy: false,
});
```