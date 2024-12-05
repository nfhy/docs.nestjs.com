### 标量

GraphQL对象类型有一个名称和字段，但这些字段最终需要解析为一些具体的数据。这就是标量类型发挥作用的地方：它们代表查询的叶子节点（了解更多[这里](https://graphql.org/learn/schema/#scalar-types)）。GraphQL包括以下默认类型：`Int`、`Float`、`String`、`Boolean`和`ID`。除了这些内置类型外，您可能还需要支持自定义的原子数据类型（例如，`Date`）。

#### 代码优先

代码优先方法提供了五个标量，其中三个是现有GraphQL类型的简单别名。

- `ID`（别名为`GraphQLID`）- 表示一个唯一标识符，通常用于重新获取一个对象或作为缓存的键
- `Int`（别名为`GraphQLInt`）- 一个有符号的32位整数
- `Float`（别名为`GraphQLFloat`）- 一个有符号的双精度浮点值
- `GraphQLISODateTime` - 一个UTC的日期时间字符串（默认用于表示`Date`类型）
- `GraphQLTimestamp` - 一个有符号整数，表示从UNIX纪元开始的日期和时间，以毫秒为单位

`GraphQLISODateTime`（例如`2019-12-03T09:54:33Z`）默认用于表示`Date`类型。要使用`GraphQLTimestamp`代替，将`buildSchemaOptions`对象的`dateScalarMode`设置为`'timestamp'`，如下所示：

```typescript
GraphQLModule.forRoot({
  buildSchemaOptions: {
    dateScalarMode: 'timestamp',
  }
}),
```

同样，`GraphQLFloat`默认用于表示`number`类型。要使用`GraphQLInt`代替，将`buildSchemaOptions`对象的`numberScalarMode`设置为`'integer'`，如下所示：

```typescript
GraphQLModule.forRoot({
  buildSchemaOptions: {
    numberScalarMode: 'integer',
  }
}),
```

此外，您可以创建自定义标量。

#### 覆盖默认标量

要为`Date`标量创建自定义实现，只需创建一个新类。

```typescript
import { Scalar, CustomScalar } from '@nestjs/graphql';
import { Kind, ValueNode } from 'graphql';

@Scalar('Date', () => Date)
export class DateScalar implements CustomScalar<number, Date> {
  description = 'Date custom scalar type';

  parseValue(value: number): Date {
    return new Date(value); // 来自客户端的值
  }

  serialize(value: Date): number {
    return value.getTime(); // 发送到客户端的值
  }

  parseLiteral(ast: ValueNode): Date {
    if (ast.kind === Kind.INT) {
      return new Date(ast.value);
    }
    return null;
  }
}
```

有了这个，将`DateScalar`注册为提供者。

```typescript
@Module({
  providers: [DateScalar],
})
export class CommonModule {}
```

现在我们可以在类中使用`Date`类型。

```typescript
@Field()
creationDate: Date;
```

#### 导入自定义标量

要使用自定义标量，导入并注册为解析器。我们将使用`graphql-type-json`包进行演示。这个npm包定义了一个`JSON` GraphQL标量类型。

首先安装包：

```bash
$ npm i --save graphql-type-json
```

安装包后，我们将自定义解析器传递给`forRoot()`方法：

```typescript
import GraphQLJSON from 'graphql-type-json';

@Module({
  imports: [
    GraphQLModule.forRoot({
      resolvers: { JSON: GraphQLJSON },
    }),
  ],
})
export class AppModule {}
```

现在我们可以在类中使用`JSON`类型。

```typescript
@Field(() => GraphQLJSON)
info: JSON;
```

对于一系列有用的标量，请查看[graphql-scalars](https://www.npmjs.com/package/graphql-scalars)包。

#### 创建自定义标量

要定义自定义标量，创建一个新的`GraphQLScalarType`实例。我们将创建一个自定义的`UUID`标量。

```typescript
const regex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

function validate(uuid: unknown): string | never {
  if (typeof uuid !== 'string' || !regex.test(uuid)) {
    throw new Error('invalid uuid');
  }
  return uuid;
}

export const CustomUuidScalar = new GraphQLScalarType({
  name: 'UUID',
  description: 'A simple UUID parser',
  serialize: (value) => validate(value),
  parseValue: (value) => validate(value),
  parseLiteral: (ast) => validate(ast.value),
});
```

我们将自定义解析器传递给`forRoot()`方法：

```typescript
@Module({
  imports: [
    GraphQLModule.forRoot({
      resolvers: { UUID: CustomUuidScalar },
    }),
  ],
})
export class AppModule {}
```

现在我们可以在类中使用`UUID`类型。

```typescript
@Field(() => CustomUuidScalar)
uuid: string;
```

#### 模式优先

要定义自定义标量（了解更多关于标量[这里](https://www.apollographql.com/docs/graphql-tools/scalars.html)），创建一个类型定义和一个专用解析器。这里（如官方文档中）我们将使用`graphql-type-json`包进行演示。这个npm包定义了一个`JSON` GraphQL标量类型。

首先安装包：

```bash
$ npm i --save graphql-type-json
```

安装包后，我们将自定义解析器传递给`forRoot()`方法：

```typescript
import GraphQLJSON from 'graphql-type-json';

@Module({
  imports: [
    GraphQLModule.forRoot({
      typePaths: ['./**/*.graphql'],
      resolvers: { JSON: GraphQLJSON },
    }),
  ],
})
export class AppModule {}
```

现在我们可以在类型定义中使用`JSON`标量：

```graphql
scalar JSON

type Foo {
  field: JSON
}
```

另一种定义标量类型的方法就是创建一个简单的类。假设我们希望用`Date`类型增强我们的模式。

```typescript
import { Scalar, CustomScalar } from '@nestjs/graphql';
import { Kind, ValueNode } from 'graphql';

@Scalar('Date')
export class DateScalar implements CustomScalar<number, Date> {
  description = 'Date custom scalar type';

  parseValue(value: number): Date {
    return new Date(value); // 来自客户端的值
  }

  serialize(value: Date): number {
    return value.getTime(); // 发送到客户端的值
  }

  parseLiteral(ast: ValueNode): Date {
    if (ast.kind === Kind.INT) {
      return new Date(ast.value);
    }
    return null;
  }
}
```

有了这个，将`DateScalar`注册为提供者。

```typescript
@Module({
  providers: [DateScalar],
})
export class CommonModule {}
```

现在我们可以在类型定义中使用`Date`标量。

```graphql
scalar Date
```

默认情况下，所有标量生成的TypeScript定义是`any` - 这并不特别类型安全。但是，您可以配置Nest生成自定义标量的类型定义时，指定如何生成类型：

```typescript
import { GraphQLDefinitionsFactory } from '@nestjs/graphql';
import { join } from 'path';

const definitionsFactory = new GraphQLDefinitionsFactory();

definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
  defaultScalarType: 'unknown',
  customScalarTypeMapping: {
    DateTime: 'Date',
    BigNumber: '_BigNumber',
  },
  additionalHeader: "import _BigNumber from 'bignumber.js'",
});
```

> 提示：或者，您可以使用类型引用，例如：`DateTime: Date`。在这种情况下，`GraphQLDefinitionsFactory`将提取指定类型的名称属性（`Date.name`）以生成TS定义。注意：添加非内置类型（自定义类型）的导入语句是必需的。

现在，给定以下GraphQL自定义标量类型：

```graphql
scalar DateTime
scalar BigNumber
scalar Payload
```

我们将在`src/graphql.ts`中看到以下生成的TypeScript定义：

```typescript
import _BigNumber from 'bignumber.js';

export type DateTime = Date;
export type BigNumber = _BigNumber;
export type Payload = unknown;
```

在这里，我们使用了`customScalarTypeMapping`属性来提供一个我们希望声明为我们的自定义标量的类型的映射。我们还提供了一个`additionalHeader`属性，以便我们可以添加这些类型定义所需的任何导入语句。最后，我们将`defaultScalarType`设置为`'unknown'`，以便任何在`customScalarTypeMapping`中未指定的自定义标量都将被别名为`unknown`而不是`any`（这是[TypeScript推荐](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-0.html#new-unknown-top-type)自3.0版本以来为了增加类型安全性而使用的）。

> 提示：请注意，我们从`bignumber.js`导入了`_BigNumber`；这是为了避免[循环类型引用](https://github.com/Microsoft/TypeScript/issues/12525#issuecomment-263166239)。