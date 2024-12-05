### CQRS

简单的[CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)（创建、读取、更新和删除）应用程序的流程可以描述如下：

1. 控制器层处理HTTP请求，并将任务委托给服务层。
2. 服务层是大部分业务逻辑所在的地方。
3. 服务使用仓库/数据访问对象来更改/持久化实体。
4. 实体作为值的容器，具有设置器和获取器。

虽然这种模式通常对小型和中型应用程序来说是足够的，但对于更大、更复杂的应用程序来说，可能不是最佳选择。在这种情况下，**CQRS**（命令和查询责任分离）模型可能更合适且可扩展（取决于应用程序的要求）。此模型的好处包括：

- **分离关注点**。该模型将读写操作分离到不同的模型中。
- **可扩展性**。读写操作可以独立扩展。
- **灵活性**。该模型允许使用不同的数据存储进行读写操作。
- **性能**。该模型允许使用针对读写操作优化的不同数据存储。

为了促进这种模型，Nest提供了一个轻量级的[CQRS模块](https://github.com/nestjs/cqrs)。本章描述了如何使用它。

#### 安装

首先安装所需的包：

```bash
$ npm install --save @nestjs/cqrs
```

#### 命令

命令用于更改应用程序状态。它们应该是基于任务的，而不是以数据为中心的。当命令被派发时，它由相应的**命令处理器**处理。处理器负责更新应用程序状态。

```typescript
@Injectable()
export class HeroesGameService {
  constructor(private commandBus: CommandBus) {}

  async killDragon(heroId: string, killDragonDto: KillDragonDto) {
    return this.commandBus.execute(
      new KillDragonCommand(heroId, killDragonDto.dragonId)
    );
  }
}
```

在上面的代码片段中，我们实例化了`KillDragonCommand`类，并将其传递给`CommandBus`的`execute()`方法。这是演示的命令类：

```typescript
export class KillDragonCommand {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {}
}
```

`CommandBus`代表一个命令流。它负责将命令派发给适当的处理器。`execute()`方法返回一个承诺，该承诺解析为处理器返回的值。

让我们为`KillDragonCommand`命令创建一个处理器。

```typescript
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(private repository: HeroRepository) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.repository.findOneById(+heroId);

    hero.killEnemy(dragonId);
    await this.repository.persist(hero);
  }
}
```

这个处理器从仓库中检索`Hero`实体，调用`killEnemy()`方法，然后持久化更改。`KillDragonHandler`类实现了`ICommandHandler`接口，需要实现`execute()`方法。`execute()`方法接收命令对象作为参数。

#### 查询

查询用于从应用程序状态中检索数据。它们应该是以数据为中心的，而不是基于任务的。当查询被派发时，它由相应的**查询处理器**处理。处理器负责检索数据。

`QueryBus`遵循与`CommandBus`相同的模式。查询处理器应该实现`IQueryHandler`接口，并用`@QueryHandler()`装饰器注解。

#### 事件

事件用于通知应用程序的其他部分关于应用程序状态的更改。它们由**模型**直接使用`EventBus`派发。当事件被派发时，它由相应的**事件处理器**处理。处理器随后可以更新读取模型。

为了演示，让我们创建一个事件类：

```typescript
export class HeroKilledDragonEvent {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {}
}
```

现在，虽然事件可以直接使用`EventBus.publish()`方法派发，我们也可以从模型中派发它们。让我们更新`Hero`模型，以便在调用`killEnemy()`方法时派发`HeroKilledDragonEvent`事件。

```typescript
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
  }

  killEnemy(enemyId: string) {
    // 业务逻辑
    this.apply(new HeroKilledDragonEvent(this.id, enemyId));
  }
}
```

`apply()`方法用于派发事件。它接受一个事件对象作为参数。然而，由于我们的模型不知道`EventBus`，我们需要将模型与`EventBus`关联起来。我们可以使用`EventPublisher`类来实现这一点。

```typescript
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(
    private repository: HeroRepository,
    private publisher: EventPublisher,
  ) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.publisher.mergeObjectContext(
      await this.repository.findOneById(+heroId),
    );
    hero.killEnemy(dragonId);
    hero.commit();
  }
}
```

`EventPublisher#mergeObjectContext`方法将事件发布者合并到提供的对象中，这意味着对象现在可以将事件发布到事件流中。

注意，在本例中我们还在模型上调用了`commit()`方法。这个方法用于派发任何未处理的事件。要自动派发事件，我们可以将`autoCommit`属性设置为`true`：

```typescript
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
    this.autoCommit = true;
  }
}
```

如果我们想要将事件发布者合并到一个不存在的对象中，而是合并到一个类中，我们可以使用`EventPublisher#mergeClassContext`方法：

```typescript
const HeroModel = this.publisher.mergeClassContext(Hero);
const hero = new HeroModel('id'); // <-- HeroModel是一个类
```

现在，`HeroModel`类的每个实例都将能够发布事件，而不需要使用`mergeObjectContext()`方法。

此外，我们可以使用`EventBus`手动发出事件：

```typescript
this.eventBus.publish(new HeroKilledDragonEvent());
```

> 信息提示：`EventBus`是一个可注入的类。

每个事件可以有多个**事件处理器**。

```typescript
@EventsHandler(HeroKilledDragonEvent)
export class HeroKilledDragonHandler implements IEventHandler<HeroKilledDragonEvent> {
  constructor(private repository: HeroRepository) {}

  handle(event: HeroKilledDragonEvent) {
    // 业务逻辑
  }
}
```

> 信息提示：请注意，当您开始使用事件处理器时，您将脱离传统的HTTP Web上下文。
>
> - `CommandHandlers`中的错误仍然可以被内置的[异常过滤器](/exception-filters)捕获。
> - `EventHandlers`中的错误不能被异常过滤器捕获：您将不得不手动处理它们。无论是通过简单的`try/catch`，使用[Sagas](/recipes/cqrs#sagas)触发补偿事件，还是您选择的任何其他解决方案。
> - `CommandHandlers`中的HTTP响应仍然可以发送回客户端。
> - `EventHandlers`中的HTTP响应不能。如果您想向客户端发送信息，您可以使用[WebSocket](/websockets/gateways)，[SSE](/techniques/server-sent-events)，或您选择的任何其他解决方案。

#### Sagas

Saga是一个长时间运行的过程，它监听事件并可能触发新的命令。它通常用于管理应用程序中的复杂工作流程。例如，当用户注册时，一个Saga可能会监听`UserRegisteredEvent`并向用户发送欢迎电子邮件。

Saga是一个非常强大的功能。一个Saga可以监听1..*事件。使用[RxJS](https://github.com/ReactiveX/rxjs)库，我们可以过滤、映射、派生和合并事件流以创建复杂的工作流程。每个Saga返回一个Observable，该Observable产生一个命令实例。然后，该命令由`CommandBus`**异步**派发。

让我们创建一个Saga，它监听`HeroKilledDragonEvent`并派发`DropAncientItemCommand`命令。

```typescript
@Injectable()
export class HeroesGameSagas {
  @Saga()
  dragonKilled = (events$: Observable<any>): Observable<ICommand> => {
    return events$.pipe(
      ofType(HeroKilledDragonEvent),
      map((event) => new DropAncientItemCommand(event.heroId, fakeItemID)),
    );
  }
}
```

> 信息提示：`ofType`操作符和`@Saga()`装饰器是从`@nestjs/cqrs`包中导出的。

`@Saga()`装饰器将方法标记为Saga。`events$`参数是一个所有事件的Observable流。`ofType`操作符通过指定的事件类型过滤流。`map`操作符将事件映射到一个新的命令实例。

在这个例子中，我们将`HeroKilledDragonEvent`映射到`DropAncientItemCommand`命令。然后，`DropAncientItemCommand`命令由`CommandBus`自动派发。

#### 设置

最后，我们需要在`HeroesGameModule`中注册所有的命令处理器、事件处理器和Sagas：

```typescript
export const CommandHandlers = [KillDragonHandler, DropAncientItemHandler];
export const EventHandlers  =  [HeroKilledDragonHandler, HeroFoundItemHandler];

@Module({
  imports: [CqrsModule],
  controllers: [HeroesGameController],
  providers: [
    HeroesGameService,
    HeroesGameSagas,
    ...CommandHandlers,
    ...EventHandlers,
    HeroRepository,
  ]
})
export class HeroesGameModule {}
```

#### 未处理的异常

事件处理器以异步方式执行。这意味着它们应该始终处理所有异常，以防止应用程序进入不一致的状态。然而，如果未处理异常，`EventBus`将创建`UnhandledExceptionInfo`对象并将其推送到`UnhandledExceptionBus`流。这个流是一个`Observable`，可以用来处理未处理的异常。

```typescript
private destroy$ = new Subject<void>();

constructor(private unhandledExceptionsBus: UnhandledExceptionBus) {
  this.unhandledExceptionsBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((exceptionInfo) => {
      // 在这里处理异常
      // 例如发送到外部服务，终止进程，或发布一个新事件
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

要过滤异常，我们可以使用`ofType`操作符，如下所示：

```typescript
this.unhandledExceptionsBus.pipe(takeUntil(this.destroy$), UnhandledExceptionBus.ofType(TransactionNotAllowedException)).subscribe((exceptionInfo) => {
  // 在这里处理异常
});
```

其中`TransactionNotAllowedException`是我们想要过滤出的异常。

`UnhandledExceptionInfo`对象包含以下属性：

```typescript
export interface UnhandledExceptionInfo<Cause = IEvent | ICommand, Exception = any> {
  /**
   * 抛出的异常。
   */
  exception: Exception;
  /**
   * 异常的原因（事件或命令引用）。
   */
  cause: Cause;
}
```

#### 订阅所有事件

`CommandBus`、`QueryBus`和`EventBus`都是**Observables**。这意味着我们可以订阅整个流，例如，处理所有事件。例如，我们可以将所有事件记录到控制台，或保存到事件存储中。

```typescript
private destroy$ = new Subject<void>();

constructor(private eventBus: EventBus) {
  this.eventBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((event) => {
      // 保存事件到数据库
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

#### 示例

一个工作示例可以在[这里](https://github.com/kamilmysliwiec/nest-cqrs-example)找到。