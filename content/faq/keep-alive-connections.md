### 保持连接

默认情况下，NestJS的HTTP适配器会等待响应完成后才关闭应用程序。但有时，这种行为并不理想，或者出乎意料。有些请求可能使用`Connection: Keep-Alive`头部，这些请求会持续很长时间。

对于这些场景，如果你总是希望你的应用程序在不等待请求结束的情况下退出，可以在创建NestJS应用程序时启用`forceCloseConnections`选项。

> 警告 **提示** 大多数用户不需要启用此选项。但需要此选项的症状是，你的应用程序不会在你预期的时候退出。通常当你启用`app.enableShutdownHooks()`并且你注意到应用程序没有重启/退出时。最有可能的情况是在开发期间使用`--watch`运行NestJS应用程序。

#### 使用方法

在你的`main.ts`文件中，在创建NestJS应用程序时启用此选项：

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    forceCloseConnections: true,
  });
  await app.listen(process.env.PORT ?? 3000);
}

bootstrap();
```