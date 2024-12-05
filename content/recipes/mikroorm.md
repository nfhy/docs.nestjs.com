### MikroORM

这篇文章旨在帮助用户在Nest中开始使用MikroORM。MikroORM是基于Data Mapper、Unit of Work和Identity Map模式的Node.js TypeScript ORM。它是TypeORM的一个绝佳替代品，从TypeORM迁移到MikroORM应该相当容易。MikroORM的完整文档可以在[这里](https://mikro-orm.io/docs)找到。

> **信息** `@mikro-orm/nestjs` 是第三方包，不是由NestJS核心团队管理。请在[适当的仓库](https://github.com/mikro-orm/nestjs)中报告与该库相关的任何问题。

#### 安装

将MikroORM集成到Nest的最简单方式是通过[`@mikro-orm/nestjs`模块](https://github.com/mikro-orm/nestjs)。
只需在Nest、MikroORM和底层驱动程序旁边安装它：

```bash
$ npm i @mikro-orm/core @mikro-orm/nestjs @mikro-orm/sqlite
```

MikroORM还支持`postgres`、`sqlite`和`mongo`。查看[官方文档](https://mikro-orm.io/docs/usage-with-sql/)了解所有驱动程序。

一旦安装过程完成，我们可以将`MikroOrmModule`导入到根`AppModule`中。

```typescript
import { SqliteDriver } from '@mikro-orm/sqlite';

@Module({
  imports: [
    MikroOrmModule.forRoot({
      entities: ['./dist/entities'],
      entitiesTs: ['./src/entities'],
      dbName: 'my-db-name.sqlite3',
      driver: SqliteDriver,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`forRoot()`方法接受与MikroORM包中的`init()`相同的配置对象。查看[此页面](https://mikro-orm.io/docs/configuration)了解完整的配置文档。

或者我们可以[配置CLI](https://mikro-orm.io/docs/installation#setting-up-the-commandline-tool)，通过创建配置文件`mikro-orm.config.ts`然后不带任何参数调用`forRoot()`。

```typescript
@Module({
  imports: [
    MikroOrmModule.forRoot(),
  ],
  ...
})
export class AppModule {}
```

但如果使用使用tree shaking的构建工具，则不会起作用，最好显式提供配置：

```typescript
import config from './mikro-orm.config'; // 你的ORM配置

@Module({
  imports: [
    MikroOrmModule.forRoot(config),
  ],
  ...
})
export class AppModule {}
```

之后，`EntityManager`将在整个项目中可供注入（无需在其他地方导入任何模块）。

```ts
// 从你的驱动包或`@mikro-orm/knex`导入一切
import { EntityManager, MikroORM } from '@mikro-orm/sqlite';

@Injectable()
export class MyService {
  constructor(
    private readonly orm: MikroORM,
    private readonly em: EntityManager,
  ) {}
}
```

> **信息** 注意`EntityManager`是从`@mikro-orm/driver`包导入的，其中driver是`mysql`、`sqlite`、`postgres`或你使用的任何驱动程序。如果你安装了`@mikro-orm/knex`作为依赖项，也可以从那里导入`EntityManager`。

#### 仓库

MikroORM支持仓库设计模式。对于每个实体，我们可以创建一个仓库。阅读有关仓库的完整文档[这里](https://mikro-orm.io/docs/repositories)。要定义哪些仓库应该在当前作用域中注册，可以使用`forFeature()`方法。例如，以这种方式：

> **信息** 你**不应该**通过`forFeature()`注册你的基实体，因为那些没有
> 仓库。另一方面，基实体需要是`forRoot()`（或ORM配置中）列表的一部分。

```typescript
// photo.module.ts
@Module({
  imports: [MikroOrmModule.forFeature([Photo])],
  providers: [PhotoService],
  controllers: [PhotoController],
})
export class PhotoModule {}
```

并将其导入到根`AppModule`中：

```typescript
// app.module.ts
@Module({
  imports: [MikroOrmModule.forRoot(...), PhotoModule],
})
export class AppModule {}
```

通过这种方式，我们可以使用`@InjectRepository()`装饰器将`PhotoRepository`注入到`PhotoService`中：

```typescript
@Injectable()
export class PhotoService {
  constructor(
    @InjectRepository(Photo)
    private readonly photoRepository: EntityRepository<Photo>,
  ) {}
}
```

#### 使用自定义仓库

当使用自定义仓库时，我们不再需要`@InjectRepository()`装饰器，因为Nest DI基于类引用解析。

```ts
// `**./author.entity.ts**`
@Entity({ repository: () => AuthorRepository })
export class Author {
  // 允许在 `em.getRepository()` 中进行推断
  [EntityRepositoryType]?: AuthorRepository;
}

// `**./author.repository.ts**`
export class AuthorRepository extends EntityRepository<Author> {
  // 你的自定义方法...
}
```

由于自定义仓库名称与`getRepositoryToken()`将返回的相同，我们不再需要`@InjectRepository()`装饰器：

```ts
@Injectable()
export class MyService {
  constructor(private readonly repo: AuthorRepository) {}
}
```

#### 自动加载实体

手动将实体添加到连接选项的实体数组中可能很繁琐。此外，从根模块引用实体会破坏应用程序领域边界，并导致实现细节泄露到应用程序的其他部分。为了解决这个问题，可以使用静态全局路径。

然而，需要注意的是，全局路径不受webpack支持，因此如果你在monorepo中构建应用程序，将无法使用它们。为了解决这个问题，提供了一个替代解决方案。要自动加载实体，将配置对象（传递给`forRoot()`方法）的`autoLoadEntities`属性设置为`true`，如下所示：

```ts
@Module({
  imports: [
    MikroOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

有了这个选项，通过`forFeature()`方法注册的每个实体将自动添加到配置对象的实体数组中。

> **信息** 注意，没有通过`forFeature()`方法注册的实体，但是
> 从实体（通过关系）引用的实体，不会被`autoLoadEntities`设置包括在内。

> **信息** 使用`autoLoadEntities`对MikroORM CLI也无效 - 对于CLI我们仍然需要具有完整实体列表的CLI配置。另一方面，我们可以
> 在CLI中使用全局路径，因为CLI不会通过webpack。

#### 序列化

> **注意** MikroORM将每个实体关系包装在`Reference<T>`或`Collection<T>`对象中，以提供更好的类型安全性。这将使[Nest的内置序列化器](/techniques/serialization)对任何包装关系视而不见。换句话说，如果你从HTTP或WebSocket处理程序返回MikroORM实体，它们的所有关系将不会被序列化。

幸运的是，MikroORM提供了一个[序列化API](https://mikro-orm.io/docs/serializing)，可以代替`ClassSerializerInterceptor`使用。

```typescript
@Entity()
export class Book {
  @Property({ hidden: true }) // 类似于class-transformer的`@Exclude`
  hiddenField = Date.now();

  @Property({ persist: false }) // 类似于class-transformer的`@Expose()`。将只存在于内存中，并将被序列化。
  count?: number;

  @ManyToOne({
    serializer: (value) => value.name,
    serializedName: 'authorName',
  }) // 类似于class-transformer的`@Transform()`
  author: Author;
}
```

#### 请求作用域处理程序在队列中

正如[文档](https://mikro-orm.io/docs/identity-map)中提到的，我们需要为每个请求提供一个干净的州。这要归功于通过中间件注册的`RequestContext`帮助器，它会自动处理。

但是中间件仅对常规HTTP请求处理执行，如果我们在外部需要请求作用域方法呢？一个例子是队列处理程序或计划任务。

我们可以使用`@CreateRequestContext()`装饰器。它需要你首先将`MikroORM`实例注入到当前上下文中，然后它将用于为你创建上下文。在底层，装饰器将为你的方法注册新的请求上下文，并在上下文中执行它。

```ts
@Injectable()
export class MyService {
  constructor(private readonly orm: MikroORM) {}

  @CreateRequestContext()
  async doSomething() {
    // 这将在单独的上下文中执行
  }
}
```

> **注意** 如名称所示，这个装饰器总是创建新的上下文，与其替代品`@EnsureRequestContext`相反，后者只有在尚未在另一个上下文中时才创建它。

#### 测试

`@mikro-orm/nestjs`包公开了`getRepositoryToken()`函数，根据给定的实体返回准备好的令牌，以允许模拟仓库。

```typescript
@Module({
  providers: [
    PhotoService,
    {
      // 或者当你有一个自定义仓库：`provide: PhotoRepository`
      provide: getRepositoryToken(Photo), 
      useValue: mockedRepository,
    },
  ],
})
export class PhotoModule {}
```

#### 示例

NestJS与MikroORM的真实世界示例可以在[这里](https://github.com/mikro-orm/nestjs-realworld-example-app)找到。