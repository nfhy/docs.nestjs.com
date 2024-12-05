### 模块引用

Nest 提供了 `ModuleRef` 类来浏览内部提供者列表，并使用其注入令牌作为查找键来获取任何提供者的引用。`ModuleRef` 类还提供了一种动态实例化静态和作用域提供者的方法。`ModuleRef` 可以像正常一样注入到类中：

```typescript
@Injectable()
export class CatsService {
  constructor(private moduleRef: ModuleRef) {}
}
```

> 信息 **提示** `ModuleRef` 类是从 `@nestjs/core` 包中导入的。

#### 检索实例

`ModuleRef` 实例（以下简称 **模块引用**）有一个 `get()` 方法。默认情况下，此方法返回在 *当前模块* 中注册并已实例化的提供者、控制器或可注入对象（例如，守卫、拦截器等），使用其注入令牌/类名。如果找不到实例，将引发异常。

```typescript
@Injectable()
export class CatsService implements OnModuleInit {
  private service: Service;
  constructor(private moduleRef: ModuleRef) {}

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
```

> 警告 **警告** 您不能使用 `get()` 方法检索作用域提供者（瞬态或请求作用域）。相反，请使用下面描述的技术。了解如何控制作用域 [在这里](/fundamentals/injection-scopes)。

要从全局上下文检索提供者（例如，如果提供者已在不同的模块中注入），请将 `{ strict: false }` 选项作为第二个参数传递给 `get()`。

```typescript
this.moduleRef.get(Service, { strict: false });
```

#### 解析作用域提供者

要动态解析作用域提供者（瞬态或请求作用域），请使用 `resolve()` 方法，将提供者的注入令牌作为参数传递。

```typescript
@Injectable()
export class CatsService implements OnModuleInit {
  private transientService: TransientService;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
```

`resolve()` 方法返回提供者的唯一实例，来自其自己的 **DI容器子树**。每个子树都有一个唯一的 **上下文标识符**。因此，如果您多次调用此方法并比较实例引用，您会发现它们不相等。

```typescript
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
```

要生成多个 `resolve()` 调用之间的单个实例，并确保它们共享相同的生成 DI 容器子树，您可以向 `resolve()` 方法传递上下文标识符。使用 `ContextIdFactory` 类生成上下文标识符。这个类提供了一个 `create()` 方法，返回一个适当的唯一标识符。

```typescript
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
```

> 信息 **提示** `ContextIdFactory` 类是从 `@nestjs/core` 包中导入的。

#### 注册 `REQUEST` 提供者

手动生成的上下文标识符（使用 `ContextIdFactory.create()`）代表 DI 子树，其中 `REQUEST` 提供者是 `undefined`，因为它们不是由 Nest 依赖注入系统实例化和管理的。

要为手动创建的 DI 子树注册自定义 `REQUEST` 对象，请使用 `ModuleRef#registerRequestByContextId()` 方法，如下所示：

```typescript
const contextId = ContextIdFactory.create();
this.moduleRef.registerRequestByContextId(/* YOUR_REQUEST_OBJECT */, contextId);
```

#### 获取当前子树

有时，您可能希望在 **请求上下文** 中解析请求作用域提供者的实例。假设 `CatsService` 是请求作用域的，您希望解析也标记为请求作用域提供者的 `CatsRepository` 实例。为了共享相同的 DI 容器子树，您必须获取当前上下文标识符，而不是生成一个新的（例如，使用上面显示的 `ContextIdFactory.create()` 函数）。要获取当前上下文标识符，请通过 `@Inject()` 装饰器注入请求对象。

```typescript
@Injectable()
export class CatsService {
  constructor(
    @Inject(REQUEST) private request: Record<string, unknown>,
  ) {}
}
```

> 信息 **提示** 了解更多关于请求提供者的信息 [在这里](https://docs.nestjs.com/fundamentals/injection-scopes#request-provider)。

现在，使用 `ContextIdFactory` 类的 `getByRequest()` 方法根据请求对象创建上下文 ID，并将此传递给 `resolve()` 调用：

```typescript
const contextId = ContextIdFactory.getByRequest(this.request);
const catsRepository = await this.moduleRef.resolve(CatsRepository, contextId);
```

#### 动态实例化自定义类

要动态实例化一个 **之前未注册为提供者** 的类，请使用模块引用的 `create()` 方法。

```typescript
@Injectable()
export class CatsService implements OnModuleInit {
  private catsFactory: CatsFactory;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
```

这种技术使您能够在框架容器之外有条件地实例化不同的类。