### 服务静态内容

为了服务静态内容，比如单页应用（SPA），我们可以使用来自 [`@nestjs/serve-static`](https://www.npmjs.com/package/@nestjs/serve-static) 包的 `ServeStaticModule`。

#### 安装

首先，我们需要安装所需的包：

```bash
$ npm install --save @nestjs/serve-static
```

#### 引导启动

安装完成后，我们可以将 `ServeStaticModule` 导入到根 `AppModule` 并配置它，通过传递一个配置对象到 `forRoot()` 方法。

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'path';

@Module({
  imports: [
    ServeStaticModule.forRoot({
      rootPath: join(__dirname, '..', 'client'),
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

有了这个配置，构建静态网站并将其内容放置在由 `rootPath` 属性指定的位置。

#### 配置

[ServeStaticModule](https://github.com/nestjs/serve-static) 可以通过多种选项进行配置，以自定义其行为。
你可以设置路径以渲染你的静态应用，指定排除路径，启用或禁用设置 Cache-Control 响应头等。查看完整选项列表 [这里](https://github.com/nestjs/serve-static/blob/master/lib/interfaces/serve-static-options.interface.ts)。

> 注意：静态应用的默认 `renderPath` 是 `*`（所有路径），模块将发送 "index.html" 文件作为响应。
> 它允许你为你的 SPA 创建客户端路由。在你的控制器中指定的路径将回落到服务器。
> 你可以通过设置 `serveRoot`、`renderPath` 结合其他选项来改变这种行为。

#### 示例

一个工作示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/24-serve-static) 查看。