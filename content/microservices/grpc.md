### gRPC

[gRPC](https://github.com/grpc/grpc-node) 是一个现代的、开源的、高性能的 RPC 框架，可以在任何环境中运行。它能够高效地连接数据中心内部和跨数据中心的服务，并可插拔地支持负载均衡、追踪、健康检查和认证。

像许多 RPC 系统一样，gRPC 基于定义服务的概念，即可以远程调用的函数（方法）。对于每个方法，您需要定义参数和返回类型。服务、参数和返回类型在 `.proto` 文件中使用 Google 的开源语言中立 [protocol buffers](https://protobuf.dev) 机制定义。

使用 gRPC 传输器，Nest 利用 `.proto` 文件动态绑定客户端和服务器，使其易于实现远程过程调用，自动序列化和反序列化结构化数据。

#### 安装

要开始构建基于 gRPC 的微服务，首先安装所需的包：

```bash
$ npm i --save @grpc/grpc-js @grpc/proto-loader
```

#### 概览

像其他 Nest 微服务传输层实现一样，您使用 `createMicroservice()` 方法传递的选项对象中的 `transport` 属性来选择 gRPC 传输器机制。以下示例中，我们将设置一个英雄服务。`options` 属性提供了有关该服务的元数据；其属性描述在[下面](microservices/grpc#options)。

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
```

> 信息 **提示** `join()` 函数从 `path` 包导入；`Transport` 枚举从 `@nestjs/microservices` 包导入。

在 `nest-cli.json` 文件中，我们添加了 `assets` 属性，允许我们分发非 TypeScript 文件，并 `watchAssets` - 打开对所有非 TypeScript 资产的监视。在我们的情况下，我们希望 `.proto` 文件自动复制到 `dist` 文件夹。

```json
{
  "compilerOptions": {
    "assets": ["**/*.proto"],
    "watchAssets": true
  }
}
```

#### 选项

gRPC 传输器选项对象暴露了以下描述的属性。

<table>
  <tr>
    <td><code>package</code></td>
    <td>Protobuf 包名称（与 <code>.proto</code> 文件中的 <code>package</code> 设置匹配）。必需</td>
  </tr>
  <tr>
    <td><code>protoPath</code></td>
    <td>
      绝对（或相对于根目录）路径到 <code>.proto</code> 文件。必需
    </td>
  </tr>
  <tr>
    <td><code>url</code></td>
    <td>连接 URL。字符串格式为 <code>ip address/dns name:port</code>（例如，Docker 服务器的 <code>'0.0.0.0:50051'</code>），定义传输器建立连接的地址/端口。可选。默认为 <code>'localhost:5000'</code></td>
  </tr>
  <tr>
    <td><code>protoLoader</code></td>
    <td>加载 <code>.proto</code> 文件的 NPM 包名称。可选。默认为 <code>'@grpc/proto-loader'</code></td>
  </tr>
  <tr>
    <td><code>loader</code></td>
    <td>
      <code>@grpc/proto-loader</code> 选项。这些提供了对 <code>.proto</code> 文件行为的详细控制。可选。更多详情见
      <a href="https://github.com/grpc/grpc-node/blob/master/packages/proto-loader/README.md" rel="nofollow" target="_blank">这里</a>
    </td>
  </tr>
  <tr>
    <td><code>credentials</code></td>
    <td>
      服务器凭据。可选。<a href="https://grpc.io/grpc/node/grpc.ServerCredentials.html" rel="nofollow" target="_blank">更多信息</a>
    </td>
  </tr>
</table>

#### 示例 gRPC 服务

我们定义一个名为 `HeroesService` 的示例 gRPC 服务。在上面的 `options` 对象中，`protoPath` 属性设置了路径到 `.proto` 定义文件 `hero.proto`。`hero.proto` 文件使用 [protocol buffers](https://developers.google.com/protocol-buffers) 结构化。

```typescript
// hero/hero.proto
syntax = "proto3";

package hero;

service HeroesService {
  rpc FindOne (HeroById) returns (Hero) {}
}

message HeroById {
  int32 id = 1;
}

message Hero {
  int32 id = 1;
  string name = 2;
}
```

我们的 `HeroesService` 暴露了一个 `FindOne()` 方法。这个方法期望一个输入参数类型为 `HeroById`，并返回一个 `Hero` 消息（protocol buffers 使用 `message` 元素定义参数类型和返回类型）。

接下来，我们需要实现服务。要定义一个满足此定义的处理程序，我们在控制器中使用 `@GrpcMethod()` 装饰器，如下所示。这个装饰器提供了声明一个方法为 gRPC 服务方法所需的元数据。

> 信息 **提示** 之前在微服务章节中介绍的 `@MessagePattern()` 装饰器（[阅读更多](microservices/basics#request-response)）不适用于基于 gRPC 的微服务。对于基于 gRPC 的微服务，`@GrpcMethod()` 装饰器有效地取代了它。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

> 信息 **提示** `@GrpcMethod()` 装饰器从 `@nestjs/microservices` 包导入，而 `Metadata` 和 `ServerUnaryCall` 从 `grpc` 包导入。

上述装饰器接受两个参数。第一个是服务名称（例如，`'HeroesService'`），对应于 `hero.proto` 中的 `HeroesService` 服务定义。第二个（字符串 `'FindOne'`）对应于 `HeroesService` 中定义的 `FindOne()` rpc 方法。

`findOne()` 处理程序方法接受三个参数，来自调用者的 `data`，存储 gRPC 请求元数据的 `metadata` 和 `call` 以获取 `GrpcCall` 对象属性，例如 `sendMetadata` 用于向客户端发送元数据。

两个 `@GrpcMethod()` 装饰器参数都是可选的。如果不带第二个参数调用（例如，`'FindOne'`），Nest 将根据将处理程序名称转换为大驼峰命名（例如，`findOne` 处理程序与 `FindOne` rpc 调用定义相关联）自动将 `.proto` 文件 rpc 方法与处理程序关联起来。如下所示。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

您还可以省略第一个 `@GrpcMethod()` 参数。在这种情况下，Nest 将根据定义处理程序的 **类** 名称自动将处理程序与 proto 定义文件中的服务定义关联起来。例如，在以下代码中，类 `HeroesService` 根据名称 `'HeroesService'` 将其处理程序方法与 `hero.proto` 文件中的 `HeroesService` 服务定义关联起来。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

#### 客户端

Nest 应用程序可以作为 gRPC 客户端，使用在 `.proto` 文件中定义的服务。您可以通过 `ClientGrpc` 对象访问远程服务。您可以以多种方式获得 `ClientGrpc` 对象。

首选技术是导入 `ClientsModule`。使用 `register()` 方法将定义在 `.proto` 文件中的服务包绑定到注入令牌，并配置服务。`name` 属性是注入令牌。对于 gRPC 服务，请使用 `transport: Transport.GRPC`。`options` 属性是一个具有上述描述的相同属性的对象。

```typescript
imports: [
  ClientsModule.register([
    {
      name: 'HERO_PACKAGE',
      transport: Transport.GRPC,
      options: {
        package: 'hero',
        protoPath: join(__dirname, 'hero/hero.proto'),
      },
    },
  ]),
];
```

> 信息 **提示** `register()` 方法接受一个对象数组。通过提供注册对象的逗号分隔列表来注册多个包。

注册后，我们可以使用 `@Inject()` 注入配置好的 `ClientGrpc` 对象。然后我们使用 `ClientGrpc` 对象的 `getService()` 方法检索服务实例，如下所示。

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  private heroesService: HeroesService;

  constructor(@Inject('HERO_PACKAGE') private client: ClientGrpc) {}

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

> 错误 **警告** gRPC 客户端不会发送名称中包含下划线 `_` 的字段，除非在 proto 加载器配置中将 `keepCase` 选项设置为 `true`（在微服务传输器配置中的 `options.loader.keepcase`）。

与在其他微服务传输方法中使用的技术相比，有一个小区别。我们不是使用 `ClientProxy` 类，而是使用 `ClientGrpc` 类，它提供了 `getService()` 方法。`getService()` 泛型方法接受一个服务名称作为参数，并返回其实例（如果可用）。

或者，您可以使用 `@Client()` 装饰器实例化一个 `ClientGrpc` 对象，如下所示：

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  @Client({
    transport: Transport.GRPC,
    options: {
      package: 'hero',
      protoPath: join(__dirname, 'hero/hero.proto'),
    },
  })
  client: ClientGrpc;

  private heroesService: HeroesService;

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

最后，对于更复杂的场景，我们可以使用 `ClientProxyFactory` 类注入动态配置的客户端，如 [这里](/microservices/basics#client) 所述。

无论哪种情况，我们最终都会得到对 `HeroesService` 代理对象的引用，它暴露了 `.proto` 文件中定义的相同一组方法。现在，当我们访问这个代理对象（即 `heroesService`）时，gRPC 系统会自动序列化请求，将它们转发到远程系统，返回响应，并反序列化响应。因为 gRPC 保护我们免受这些网络通信细节的影响，`heroesService` 看起来和行为就像一个本地提供者。

注意，所有服务方法都是 **小驼峰命名**（为了遵循语言的自然约定）。因此，例如，虽然我们的 `.proto` 文件 `HeroesService` 定义包含 `FindOne()` 函数，`heroesService` 实例将提供 `findOne()` 方法。

```typescript
interface HeroesService {
  findOne(data: { id: number }): Observable<any>;
}
```

消息处理程序也能够返回一个 `Observable`，在这种情况下，结果值将被发出，直到流完成。

```typescript
@@filename(heroes.controller)
@Get()
call(): Observable<any> {
  return this.heroesService.findOne({ id: 1 });
}
@@switch
@Get()
call() {
  return this.heroesService.findOne({ id: 1 });
}
```

要发送 gRPC 元数据（随请求一起），可以传递第二个参数，如下：

```typescript
call(): Observable<any> {
  const metadata = new Metadata();
  metadata.add('Set-Cookie', 'yummy_cookie=choco');

  return this.heroesService.findOne({ id: 1 }, metadata);
}
```

> 信息 **提示** `Metadata` 类从 `grpc` 包导入。

请注意，这将需要更新我们之前定义的 `HeroesService` 接口。

#### 示例

一个工作示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/04-grpc) 找到。

#### gRPC 反射

[gRPC 服务器反射规范](https://grpc.io/docs/guides/reflection/#overview) 是一个标准，允许 gRPC 客户端请求服务器公开的 API 详细信息，类似于为 REST API 公开 OpenAPI 文档。这可以使使用开发者调试工具（如 grpc-ui 或 postman）变得更加容易。

要将 gRPC 反射支持添加到您的服务器，请首先安装所需的实现包：

```bash
$ npm i --save @grpc/reflection
```

然后，您可以在 gRPC 服务器选项中的 `onLoadPackageDefinition` 钩子中将其挂接到 gRPC 服务器，如下所示：

```typescript
@@filename(main)
import { ReflectionService } from '@grpc/reflection';

const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  options: {
    onLoadPackageDefinition: (pkg, server) => {
      new ReflectionService(pkg).addToServer(server);
    },
  },
});
```

现在，您的服务器将使用反射规范响应请求 API 详细信息的消息。

#### gRPC 流

gRPC 本身支持长期实时连接，通常称为 `流`。流对于聊天、观察或块数据传输等场景非常有用。在官方文档 [这里](https://grpc.io/docs/guides/concepts/) 了解更多详情。

Nest 以两种可能的方式支持 GRPC 流处理程序：

- RxJS `Subject` + `Observable` 处理程序：可用于在控制器方法内编写响应或将其传递给 `Subject`/`Observable` 消费者
- 纯 GRPC 调用流处理程序：可用于传递给某个执行器，该执行器将处理 Node 标准 `Duplex` 流处理程序的其余分派。

<app-banner-enterprise></app-banner-enterprise>

#### 流示例

让我们定义一个新的示例 gRPC 服务，称为 `HelloService`。`hello.proto` 文件使用 [protocol buffers](https://developers.google.com/protocol-buffers) 结构化。它看起来像这样：

```typescript
// hello/hello.proto
syntax = "proto3";

package hello;

service HelloService {
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

> 信息 **提示** `LotsOfGreetings` 方法可以简单地使用 `@GrpcMethod` 装饰器实现（如上例所示），因为返回的流可以发出多个值。

基于此 `.proto` 文件，我们定义 `HelloService` 接口：

```typescript
interface HelloService {
  bidiHello(upstream: Observable<HelloRequest>): Observable<HelloResponse>;
  lotsOfGreetings(
    upstream: Observable<HelloRequest>,
  ): Observable<HelloResponse>;
}

interface HelloRequest {
  greeting: string;
}

interface HelloResponse {
  reply: string;
}
```

> 信息 **提示** proto 接口可以由 [ts-proto](https://github.com/stephenh/ts-proto) 包自动生成，了解更多 [这里](https://github.com/stephenh/ts-proto/blob/main/NESTJS.markdown)。

#### Subject 策略

`@GrpcStreamMethod()` 装饰器提供函数参数作为 RxJS `Observable`。因此，我们可以接收和处理多个消息。

```typescript
@GrpcStreamMethod()
bidiHello(messages: Observable<any>, metadata: Metadata, call: ServerDuplexStream<any, any>): Observable<any> {
  const subject = new Subject();

  const onNext = message => {
    console.log(message);
    subject.next({
      reply: 'Hello, world!'
    });
  };
  const onComplete = () => subject.complete();
  messages.subscribe({
    next: onNext,
    complete: onComplete,
  });

  return subject.asObservable();
}
```

> 警告 **警告** 为了支持 `@GrpcStreamMethod()` 装饰器的全双工交互，控制器方法必须返回一个 RxJS `Observable`。

> 信息 **提示** `Metadata` 和 `ServerUnaryCall` 类/接口从 `grpc` 包导入。

根据服务定义（在 `.proto` 文件中），`BidiHello` 方法应该将请求流式传输到服务。要从客户端向流发送多个异步消息，我们利用 RxJS `ReplaySubject` 类。

```typescript
const helloService = this.client.getService<HelloService>('HelloService');
const helloRequest$ = new ReplaySubject<HelloRequest>();

helloRequest$.next({ greeting: 'Hello (1)!' });
helloRequest$.next({ greeting: 'Hello (2)!' });
helloRequest$.complete();

return helloService.bidiHello(helloRequest$);
```

在上面的例子中，我们向流写入了两条消息（`next()` 调用）并通知服务我们已经完成了数据发送（`complete()` 调用）。

#### 调用流处理程序

当方法返回值被定义为 `stream` 时，`@GrpcStreamCall()` 装饰器提供函数参数作为 `grpc.ServerDuplexStream`，它支持标准方法，如 `.on('data', callback)`、`.write(message)` 或 `.cancel()`。有关可用方法的完整文档，可在 [这里](https://grpc.github.io/grpc/node/grpc-ClientDuplexStream.html) 找到。

或者，当方法返回值不是 `stream` 时，`@GrpcStreamCall()` 装饰器提供两个函数参数，分别是 `grpc.ServerReadableStream`（阅读更多 [这里](https://grpc.github.io/grpc/node/grpc-ServerReadableStream.html)）和 `callback`。

让我们开始实现 `BidiHello`，它应该支持全双工交互。

```typescript
@GrpcStreamCall()
bidiHello(requestStream: any) {
  requestStream.on('data', message => {
    console.log(message);
    requestStream.write({
      reply: 'Hello, world!'
    });
  });
}
```

> 信息 **提示** 此装饰器不需要提供任何特定的返回参数。预期流将类似于任何其他标准流类型进行处理。

在上面的例子中，我们使用 `write()` 方法向响应流写入对象。传递给 `.on()` 方法的回调作为第二个参数，将在我们的服务每次接收到新数据块时被调用。

让我们实现 `LotsOfGreetings` 方法。

```typescript
@GrpcStreamCall()
lotsOfGreetings(requestStream: any, callback: (err: unknown, value: HelloResponse) => void) {
  requestStream.on('data', message => {
    console.log(message);
  });
  requestStream.on('end', () => callback(null, { reply: 'Hello, world!' }));
}
```

在这里，我们使用 `callback` 函数在 `requestStream` 的处理完成后发送响应。

#### gRPC 元数据

元数据是关于特定 RPC 调用的信息，以键值对列表的形式存在，其中键是字符串，值通常是字符串但可以是二进制数据。元数据对 gRPC 本身是不透明的 - 它允许客户端向服务器提供与调用相关的信息，反之亦然。元数据可能包括身份验证令牌、请求标识符和用于监控目的的标签，以及数据信息，如数据集中的记录数。

在 `@GrpcMethod()` 处理程序中读取元数据，使用第二个参数（metadata），它是 `Metadata` 类型（从 `grpc` 包导入）。

要从处理程序发送回元数据，使用 `ServerUnaryCall#sendMetadata()` 方法（第三个处理程序参数）。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
```

同样，在用 `@GrpcStreamMethod()` 处理程序（[subject strategy](microservices/grpc#subject-strategy)）注解的处理程序中读取元数据，使用第二个参数（metadata），它是 `Metadata` 类型（从 `grpc` 包导入）。

要从处理程序发送回元数据，使用 `ServerDuplexStream#sendMetadata()` 方法（第三个处理程序参数）。

在 [call stream handlers](microservices/grpc#call-stream-handler)（用 `@GrpcStreamCall()` 装饰器注解的处理程序）中读取元数据，监听 `requestStream` 引用上的 `metadata` 事件，如下：

```typescript
requestStream.on('metadata', (metadata: Metadata) => {
  const meta = metadata.get('X-Meta');
});
```