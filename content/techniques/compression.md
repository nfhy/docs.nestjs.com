### 压缩

压缩可以大幅减小响应体的大小，从而提高Web应用的速度。

对于生产环境中的**高流量**网站，强烈建议将压缩任务从应用服务器卸载到反向代理中（例如，Nginx）。在这种情况下，您不应使用压缩中间件。

#### 在Express中使用（默认）

使用[compression](https://github.com/expressjs/compression)中间件包来启用gzip压缩。

首先安装所需的包：

```bash
$ npm i --save compression
```

安装完成后，将压缩中间件作为全局中间件应用。

```typescript
import * as compression from 'compression';
// 在初始化文件的某个位置
app.use(compression());
```

#### 在Fastify中使用

如果使用`FastifyAdapter`，您将需要使用[fastify-compress](https://github.com/fastify/fastify-compress)：

```bash
$ npm i --save @fastify/compress
```

安装完成后，将`@fastify/compress`中间件作为全局中间件应用。

```typescript
import compression from '@fastify/compress';
// 在初始化文件的某个位置
await app.register(compression);
```

默认情况下，`@fastify/compress`将在浏览器支持编码时使用Brotli压缩（Node >= 11.7.0）。虽然Brotli在压缩比方面可以非常高效，但它也可能非常慢。默认情况下，Brotli设置的最大压缩质量为11，尽管可以通过调整`BROTLI_PARAM_QUALITY`在0（最小）和11（最大）之间来减少压缩时间，以换取压缩质量。这将需要微调以优化空间/时间性能。质量为4的示例：

```typescript
import { constants } from 'zlib';
// 在初始化文件的某个位置
await app.register(compression, { brotliOptions: { params: { [constants.BROTLI_PARAM_QUALITY]: 4 } } });
```

为了简化，您可能希望告诉`fastify-compress`只使用deflate和gzip来压缩响应；您最终得到的响应可能会更大，但它们会被更快地传输。

要指定编码，向`app.register`提供第二个参数：

```typescript
await app.register(compression, { encodings: ['gzip', 'deflate'] });
```

上述代码告诉`fastify-compress`只使用gzip和deflate编码，如果客户端同时支持两者，则优先使用gzip。