### 映射类型

> 警告 **警告** 本章节仅适用于代码优先方法。

在构建 CRUD（创建/读取/更新/删除）等功能时，通常需要基于基本实体类型构建变体。Nest 提供了几个实用函数，执行类型转换以使这项任务更加方便。

#### 部分
在构建输入验证类型（也称为数据传输对象或 DTO）时，通常需要构建相同类型的**创建**和**更新**变体。例如，**创建**变体可能需要所有字段，而**更新**变体可能使所有字段可选。

Nest 提供了 `PartialType()` 实用函数，使这项任务更简单，减少样板代码。

`PartialType()` 函数返回一个类型（类），将输入类型的所有属性设置为可选。例如，假设我们有一个**创建**类型如下：

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

默认情况下，所有这些字段都是必需的。要创建一个具有相同字段但每个字段都是可选的类型，使用 `PartialType()` 传递类引用（`CreateUserInput`）作为参数：

```typescript
@InputType()
export class UpdateUserInput extends PartialType(CreateUserInput) {}
```

> 提示 **提示** `PartialType()` 函数从 `@nestjs/graphql` 包导入。

`PartialType()` 函数接受一个可选的第二个参数，这是一个装饰器工厂的引用。这个参数可以用来改变应用于结果（子）类的装饰器函数。如果没有指定，子类实际上使用与**父**类（第一个参数引用的类）相同的装饰器。在上面的例子中，我们扩展了用 `@InputType()` 装饰器注解的 `CreateUserInput`。由于我们希望 `UpdateUserInput` 也被视为用 `@InputType()` 装饰，我们不需要传递 `InputType` 作为第二个参数。如果父类和子类类型不同（例如，父类用 `@ObjectType` 装饰），我们会传递 `InputType` 作为第二个参数。例如：

```typescript
@InputType()
export class UpdateUserInput extends PartialType(User, InputType) {}
```

#### 选择
`PickType()` 函数通过从输入类型中选择一组属性来构建新类型（类）。例如，假设我们从以下类型开始：

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

我们可以使用 `PickType()` 实用函数从这个类中选择一组属性：

```typescript
@InputType()
export class UpdateEmailInput extends PickType(CreateUserInput, [
  'email',
] as const) {}
```

> 提示 **提示** `PickType()` 函数从 `@nestjs/graphql` 包导入。

#### 省略
`OmitType()` 函数通过从输入类型中选择所有属性，然后移除特定一组键来构建类型。例如，假设我们从以下类型开始：

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

我们可以生成一个派生类型，它具有除 `email` 之外的所有属性，如下所示。在这个构造中，`OmitType` 的第二个参数是一个属性名数组。

```typescript
@InputType()
export class UpdateUserInput extends OmitType(CreateUserInput, [
  'email',
] as const) {}
```

> 提示 **提示** `OmitType()` 函数从 `@nestjs/graphql` 包导入。

#### 交集
`IntersectionType()` 函数将两个类型合并为一个新类型（类）。例如，假设我们从两个类型开始：

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;
}

@ObjectType()
export class AdditionalUserInfo {
  @Field()
  firstName: string;

  @Field()
  lastName: string;
}
```

我们可以生成一个新类型，它结合了两种类型的所有属性。

```typescript
@InputType()
export class UpdateUserInput extends IntersectionType(
  CreateUserInput,
  AdditionalUserInfo,
) {}
```

> 提示 **提示** `IntersectionType()` 函数从 `@nestjs/graphql` 包导入。

#### 组合
类型映射实用函数是可组合的。例如，以下将产生一个类型（类），它具有 `CreateUserInput` 类型的所有属性，除了 `email`，并且这些属性将被设置为可选：

```typescript
@InputType()
export class UpdateUserInput extends PartialType(
  OmitType(CreateUserInput, ['email'] as const),
) {}
```