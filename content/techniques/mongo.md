### Mongo

Nest支持两种与[MongoDB](https://www.mongodb.com/)数据库集成的方法。您可以使用内置的[TypeORM](https://github.com/typeorm/typeorm)模块，该模块有一个MongoDB的连接器，或者使用[Mongoose](https://mongoosejs.com)，这是最受欢迎的MongoDB对象建模工具。在本章中，我们将描述后者，使用专门的`@nestjs/mongoose`包。

首先，安装[所需的依赖项](https://github.com/Automattic/mongoose)：

```bash
$ npm i @nestjs/mongoose mongoose
```

安装过程完成后，我们可以将`MongooseModule`导入根`AppModule`。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost/nest')],
})
export class AppModule {}
```

`forRoot()`方法接受与Mongoose包中的`mongoose.connect()`相同的配置对象，如[这里](https://mongoosejs.com/docs/connections.html)所述。

#### 模型注入

使用Mongoose时，一切都是从[Schema](http://mongoosejs.com/docs/guide.html)派生的。每个模式映射到MongoDB中的一个集合，并定义该集合中文档的形状。模式用于定义[模型](https://mongoosejs.com/docs/models.html)。模型负责从底层MongoDB数据库中创建和读取文档。

可以使用NestJS装饰器或手动使用Mongoose本身创建模式。使用装饰器创建模式大大减少了样板代码，并提高了代码的整体可读性。

让我们定义`CatSchema`：

```typescript
@@filename(schemas/cat.schema)
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export type CatDocument = HydratedDocument<Cat>;

@Schema()
export class Cat {
  @Prop()
  name: string;

  @Prop()
  age: number;

  @Prop()
  breed: string;
}

export const CatSchema = SchemaFactory.createForClass(Cat);
```

> 信息 **提示** 请注意，您还可以使用`DefinitionsFactory`类（来自`nestjs/mongoose`）。这允许您手动修改基于您提供的元数据生成的模式定义。这在某些边缘情况中很有用，其中可能很难用装饰器表示所有内容。

`@Schema()`装饰器将一个类标记为模式定义。它将我们的`Cat`类映射到同名的MongoDB集合，但末尾有一个额外的“s” - 因此最终的mongo集合名称将是`cats`。这个装饰器接受一个可选的参数，即模式选项对象。将其视为您通常会作为`mongoose.Schema`类的第二个参数传递的对象（例如，`new mongoose.Schema(_, options)`）。要了解更多关于可用模式选项的信息，请参见[本章](https://mongoosejs.com/docs/guide.html#options)。

`@Prop()`装饰器在文档中定义了一个属性。例如，在上述模式定义中，我们定义了三个属性：`name`、`age`和`breed`。这些属性的[模式类型](https://mongoosejs.com/docs/schematypes.html)得益于TypeScript元数据（和反射）功能，可以自动推断。然而，在类型不能隐式反射的更复杂场景中（例如，数组或嵌套对象结构），必须明确指示类型，如下所示：

```typescript
@Prop([String])
tags: string[];
```

或者，`@Prop()`装饰器接受一个选项对象参数（[了解更多](https://mongoosejs.com/docs/schematypes.html#schematype-options)关于可用选项）。通过这个，您可以指示一个属性是否必需，指定默认值，或将其标记为不可变。例如：

```typescript
@Prop({ required: true })
name: string;
```

如果您想指定与另一个模型的关系，稍后用于填充，也可以使用`@Prop()`装饰器。例如，如果`Cat`有`Owner`存储在名为`owners`的不同集合中，属性应该有类型和ref。例如：

```typescript
import * as mongoose from 'mongoose';
import { Owner } from '../owners/schemas/owner.schema';

// 在类定义中
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;
```

如果有多个所有者，您的属性配置应该如下所示：

```typescript
@Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' }] })
owners: Owner[];
```

最后，**原始**模式定义也可以传递给装饰器。这在例如属性表示未定义为类的嵌套对象时很有用。为此，请使用`@nestjs/mongoose`包中的`raw()`函数，如下所示：

```typescript
@Prop(raw({
  firstName: { type: String },
  lastName: { type: String }
}))
details: Record<string, any>;
```

或者，如果您更喜欢**不使用装饰器**，可以手动定义模式。例如：

```typescript
export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

`cat.schema`文件位于`cats`目录中的一个文件夹中，我们还在其中定义了`CatsModule`。虽然您可以根据个人喜好将模式文件存储在任何位置，但我们建议将它们存储在适当的模块目录中，靠近它们相关的**域**对象。

让我们看看`CatsModule`：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { Cat, CatSchema } from './schemas/cat.schema';

@Module({
  imports: [MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }])]
})
export class CatsModule {}
```

`MongooseModule`提供了`forFeature()`方法来配置模块，包括定义当前范围内应该注册哪些模型。如果您还想在另一个模块中使用这些模型，请将MongooseModule添加到`CatsModule`的`exports`部分，并在另一个模块中导入`CatsModule`。

注册模式后，您可以使用`@InjectModel()`装饰器将`Cat`模型注入到`CatsService`中：

```typescript
@@filename(cats.service)
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name) private catModel: Model<Cat>) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
@@switch
import { Model } from 'mongoose';
import { Injectable, Dependencies } from '@nestjs/common';
import { getModelToken } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';

@Injectable()
@Dependencies(getModelToken(Cat.name))
export class CatsService {
  constructor(catModel) {
    this.catModel = catModel;
  }

  async create(createCatDto) {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll() {
    return this.catModel.find().exec();
  }
}
```

#### 连接

有时您可能需要访问原生的[Mongoose Connection](https://mongoosejs.com/docs/api.html#Connection)对象。例如，您可能希望在连接对象上进行原生API调用。您可以使用`@InjectConnection()`装饰器注入Mongoose连接，如下所示：

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private connection: Connection) {}
}
```

#### 会话

要使用Mongoose开始会话，建议注入数据库连接使用`@InjectConnection`而不是直接调用`mongoose.startSession()`。这种方法允许更好地集成NestJS依赖注入系统，确保适当的连接管理。

以下是如何开始会话的示例：

```typescript
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private readonly connection: Connection) {}

  async startTransaction() {
    const session = await this.connection.startSession();
    session.startTransaction();
    // 您的事务逻辑在这里
  }
}
```

在这个例子中，`@InjectConnection()`用于将Mongoose连接注入到服务中。一旦注入了连接，您可以使用`connection.startSession()`开始一个新的会话。这个会话可以用来管理数据库事务，确保跨多个查询的原子操作。开始会话后，记得根据您的逻辑提交或中止事务。

#### 多个数据库

一些项目需要多个数据库连接。也可以使用此模块实现。要使用多个连接，首先创建连接。在这种情况下，连接命名成为**强制性**的。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionName: 'cats',
    }),
    MongooseModule.forRoot('mongodb://localhost/users', {
      connectionName: 'users',
    }),
  ],
})
export class AppModule {}
```

> 警告 **注意** 请注意，您不应该有无名称的多个连接，或者具有相同名称的连接，否则它们将被覆盖。

有了这个设置，您必须告诉`MongooseModule.forFeature()`函数应该使用哪个连接。

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }], 'cats'),
  ],
})
export class CatsModule {}
```

您还可以为给定连接注入`Connection`：

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection('cats') private connection: Connection) {}
}
```

要将给定的`Connection`注入到自定义提供程序（例如，工厂提供程序）中，请使用`getConnectionToken()`函数并传递连接的名称作为参数。

```typescript
{
  provide: CatsService,
  useFactory: (catsConnection: Connection) => {
    return new CatsService(catsConnection);
  },
  inject: [getConnectionToken('cats')],
}
```

如果您只是想要从一个命名数据库中注入模型，可以使用连接名称作为第二个参数到`@InjectModel()`装饰器。

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name, 'cats') private catModel: Model<Cat>) {}
}
@@switch
@Injectable()
@Dependencies(getModelToken(Cat.name, 'cats'))
export class CatsService {
  constructor(catModel) {
    this.catModel = catModel;
  }
}
```

#### 钩子（中间件）

中间件（也称为预处理和后处理钩子）是在异步函数执行期间传递控制的函数。中间件在模式级别指定，并且对于编写插件（[来源](https://mongoosejs.com/docs/middleware.html)）很有用。在编译模型后调用`pre()`或`post()`在Mongoose中不起作用。要在模型注册之前注册一个钩子，使用`MongooseModule`的`forFeatureAsync()`方法以及工厂提供程序（即，`useFactory`）。有了这种技术，您可以访问模式对象，然后使用`pre()`或`post()`方法在该模式上注册一个钩子。看下面的示例：

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.pre('save', function () {
            console.log('Hello from pre save');
          });
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

像其他[工厂提供程序](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)一样，我们的工厂函数可以是`async`，并且可以通过`inject`注入依赖项。

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => {
          const schema = CatsSchema;
          schema.pre('save', function() {
            console.log(
              `${configService.get<string>('APP_NAME')}: Hello from pre save`,
            );
          });
          return schema;
        },
        inject: [ConfigService],
      },
    ]),
  ],
})
export class AppModule {}
```

#### 插件

要为给定模式注册[插件](https://mongoosejs.com/docs/plugins.html)，请使用`forFeatureAsync()`方法。

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.plugin(require('mongoose-autopopulate'));
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

要为所有模式一次性注册插件，请在`Connection`对象上调用`.plugin()`方法。您应该在创建模型之前访问连接；为此，请使用`connectionFactory`：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionFactory: (connection) => {
        connection.plugin(require('mongoose-autopopulate'));
        return connection;
      }
    }),
  ],
})
export class AppModule {}
```

#### 鉴别器

[鉴别器](https://mongoosejs.com/docs/discriminators.html)是模式继承机制。它们使您能够在相同的MongoDB集合上拥有多个具有重叠模式的模型。

假设您想要在单个集合中跟踪不同类型的事件。每个事件都将有一个时间戳。

```typescript
@@filename(event.schema)
@Schema({ discriminatorKey: 'kind' })
export class Event {
  @Prop({
    type: String,
    required: true,
    enum: [ClickedLinkEvent.name, SignUpEvent.name],
  })
  kind: string;

  @Prop({ type: Date, required: true })
  time: Date;
}

export const EventSchema = SchemaFactory.createForClass(Event);
```

> 信息 **提示** Mongoose区分不同的鉴别器模型的方式是通过“鉴别器键”，默认为“__t”。Mongoose在您的模式中添加了一个名为“__t”的字符串路径，用于跟踪此文档是哪种鉴别器的实例。
> 您还可以使用`discriminatorKey`选项来定义歧视的路径。

`SignedUpEvent`和`ClickedLinkEvent`实例将存储在与通用事件相同的集合中。

现在，让我们定义`ClickedLinkEvent`类，如下所示：

```typescript
@@filename(click-link-event.schema)
@Schema()
export class ClickedLinkEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  url: string;
}

export const ClickedLinkEventSchema = SchemaFactory.createForClass(ClickedLinkEvent);
```

和`SignUpEvent`类：

```typescript
@@filename(sign-up-event.schema)
@Schema()
export class SignUpEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  user: string;
}

export const SignUpEventSchema = SchemaFactory.createForClass(SignUpEvent);
```

有了这个，使用`discriminators`选项为给定模式注册一个鉴别器。它在`MongooseModule.forFeature`和`MongooseModule.forFeatureAsync`上都有效：

```typescript
@@filename(event.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: Event.name,
        schema: EventSchema,
        discriminators: [
          { name: ClickedLinkEvent.name, schema: ClickedLinkEventSchema },
          { name: SignUpEvent.name, schema: SignUpEventSchema },
        ],
      },
    ]),
  ]
})
export class EventsModule {}
```

#### 测试

当单元测试应用程序时，我们通常希望避免任何数据库连接，使我们的测试套件更简单、更快地设置和执行。但我们的类可能依赖于从连接实例中提取的模型。我们如何解决这些类？解决方案是创建模拟模型。

为此更容易，`@nestjs/mongoose`包公开了一个`getModelToken()`函数，它返回一个基于令牌名称的准备[注入令牌](https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals)。使用此令牌，您可以使用任何标准的[自定义提供程序](/fundamentals/custom-providers)技术轻松提供模拟实现，包括`useClass`、`useValue`和`useFactory`。例如：

```typescript
@Module({
  providers: [
    CatsService,
    {
      provide: getModelToken(Cat.name),
      useValue: catModel,
    },
  ],
})
export class CatsModule {}
```

在这个例子中，一个硬编码的`catModel`（对象实例）将在任何消费者使用`@InjectModel()`装饰器注入`Model<Cat>`时提供。

<app-banner-courses></app-banner-courses>

#### 异步配置

当您需要异步传递模块选项而不是静态传递时，请使用`forRootAsync()`方法。与大多数动态模块一样，Nest提供了几种技术来处理异步配置。

一种技术是使用工厂函数：

```typescript
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/nest',
  }),
});
```

像其他[工厂提供程序](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)一样，我们的工厂函数可以是`async`，并且可以通过`inject`注入依赖项。

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    uri: configService.get<string>('MONGODB_URI'),
  }),
  inject: [ConfigService],
});
```

或者，您可以使用类而不是工厂来配置`MongooseModule`，如下所示：

```typescript
MongooseModule.forRootAsync({
  useClass: MongooseConfigService,
});
```

上述构造在`MongooseModule`内实例化`MongooseConfigService`，使用它来创建所需的选项对象。请注意，在这种情况下，`MongooseConfigService`必须实现`MongooseOptionsFactory`接口，如下所示。`MongooseModule`将在提供的类的实例上调用`createMongooseOptions()`方法。

```typescript
@Injectable()
export class MongooseConfigService implements MongooseOptionsFactory {
  createMongooseOptions(): MongooseModuleOptions {
    return {
      uri: 'mongodb://localhost/nest',
    };
  }
}
```

如果您想重用现有的选项提供程序，而不是在`MongooseModule`内创建一个私有副本，请使用`useExisting`语法。

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

#### 连接事件

您可以通过使用`onConnectionCreate`配置选项来监听Mongoose[连接事件](https://mongoosejs.com/docs/connections.html#connection-events)。这使您能够实现自定义逻辑，每当建立连接时。例如，您可以为`connected`、`open`、`disconnected`、`reconnected`和`disconnecting`事件注册事件侦听器，如下所示：

```typescript
MongooseModule.forRoot('mongodb://localhost/test', {
  onConnectionCreate: (connection: Connection) => {
    connection.on('connected', () => console.log('connected'));
    connection.on('open', () => console.log('open'));
    connection.on('disconnected', () => console.log('disconnected'));
    connection.on('reconnected', () => console.log('reconnected'));
    connection.on('disconnecting', () => console.log('disconnecting'));

    return connection;
  },
}),
```

在这段代码片段中，我们正在建立到`mongodb://localhost/test`的MongoDB数据库的连接。`onConnectionCreate`选项使您能够为监控连接状态设置特定的事件侦听器：

- `connected`：当连接成功建立时触发。
- `open`：当连接完全打开并准备好操作时触发。
- `disconnected`：当连接丢失时调用。
- `reconnected`：当连接在断开连接后重新建立时调用。
- `disconnecting`：当连接正在关闭时发生。

您还可以将`onConnectionCreate`属性纳入使用`MongooseModule.forRootAsync()`创建的异步配置中：

```typescript
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/test',
    onConnectionCreate: (connection: Connection) => {
      // 在这里注册事件侦听器
      return connection;
    },
  }),
}),
```

这为您提供了一种灵活的方式来管理连接事件，使您能够有效地处理连接状态的变化。

#### 子文档

要在父文档中嵌套子文档，您可以如下定义模式：

```typescript
@@filename(name.schema)
@Schema()
export class Name {
  @Prop()
  firstName: string;

  @Prop()
  lastName: string;
}

export const NameSchema = SchemaFactory.createForClass(Name);
```

然后在父模式中引用子文档：

```typescript
@@filename(person.schema)
@Schema()
export class Person {
  @Prop(NameSchema)
  name: Name;
}

export const PersonSchema = SchemaFactory.createForClass(Person);

export type PersonDocumentOverride = {
  name: Types.Subdocument<Types.ObjectId & Name>;
};

export type PersonDocument = HydratedDocument<Person, PersonDocumentOverride>;
```

如果您想包含多个子文档，可以使用子文档数组。重要的是要相应地覆盖属性的类型：

```typescript
@@filename(name.schema)
@Schema()
export class Person {
  @Prop([NameSchema])
  name: Name[];
}

export const PersonSchema = SchemaFactory.createForClass(Person);

export type PersonDocumentOverride = {
  name: Types.DocumentArray<Name>;
};

export type PersonDocument = HydratedDocument<Person, PersonDocumentOverride>;
```

#### 虚拟

在Mongoose中，**虚拟**是存在于文档上但不持久化到MongoDB的属性。它不存储在数据库中，但在访问时动态计算。虚拟通常用于派生或计算值，如组合字段（例如，通过连接`firstName`和`lastName`创建`fullName`属性），或创建依赖于文档中现有数据的属性。

```ts
class Person {
  @Prop()
  firstName: string;

  @Prop()
  lastName: string;

  @Virtual({
    get: function (this: Person) {
      return `${this.firstName} ${this.lastName}`;
    },
  })
  fullName: string;
}
```

> 信息 **提示** `@Virtual()`装饰器从`@nestjs/mongoose`包导入。

在这个例子中，`fullName`虚拟是从`firstName`和`lastName`派生的。即使它在访问时表现得像一个普通属性，但它永远不会保存到MongoDB文档中。