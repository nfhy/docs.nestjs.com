### HTTPS

要创建一个使用HTTPS协议的应用程序，在传递给`NestFactory`类的`create()`方法的选项对象中设置`httpsOptions`属性：

```typescript
const httpsOptions = {
  key: fs.readFileSync('./secrets/private-key.pem'),
  cert: fs.readFileSync('./secrets/public-certificate.pem'),
};
const app = await NestFactory.create(AppModule, {
  httpsOptions,
});
await app.listen(process.env.PORT ?? 3000);
```

如果您使用`FastifyAdapter`，请按照以下方式创建应用程序：

```typescript
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter({ https: httpsOptions }),
);
```

#### 多个同时监听的服务器

以下示例展示了如何实例化一个同时监听多个端口（例如，非HTTPS端口和HTTPS端口）的Nest应用程序。

```typescript
const httpsOptions = {
  key: fs.readFileSync('./secrets/private-key.pem'),
  cert: fs.readFileSync('./secrets/public-certificate.pem'),
};

const server = express();
const app = await NestFactory.create(AppModule, new ExpressAdapter(server));
await app.init();

const httpServer = http.createServer(server).listen(3000);
const httpsServer = https.createServer(httpsOptions, server).listen(443);
```

因为我们自己调用了`http.createServer` / `https.createServer`，所以在调用`app.close` / 终止信号时NestJS不会关闭它们。我们需要自己来做：

```typescript
@Injectable()
export class ShutdownObserver implements OnApplicationShutdown {
  private httpServers: http.Server[] = [];

  public addHttpServer(server: http.Server): void {
    this.httpServers.push(server);
  }

  public async onApplicationShutdown(): Promise<void> {
    await Promise.all(
      this.httpServers.map(
        (server) =>
          new Promise((resolve, reject) => {
            server.close((error) => {
              if (error) {
                reject(error);
              } else {
                resolve(null);
              }
            });
          }),
      ),
    );
  }
}

const shutdownObserver = app.get(ShutdownObserver);
shutdownObserver.addHttpServer(httpServer);
shutdownObserver.addHttpServer(httpsServer);
```

> 信息 **提示** `ExpressAdapter` 从 `@nestjs/platform-express` 包中导入。`http` 和 `https` 包是原生的Node.js包。

> **警告** 这个示例不适用于 [GraphQL Subscriptions](/graphql/subscriptions)。