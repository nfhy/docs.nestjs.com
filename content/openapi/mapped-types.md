### 映射类型

在构建如 **CRUD**（创建/读取/更新/删除）这样的功能时，经常需要构建基于基本实体类型的变体。Nest 提供了几个实用函数，执行类型转换，使这项任务更加方便。

#### 部分类型（Partial）

在构建输入验证类型（也称为 DTOs）时，通常需要构建同一类型的 **创建** 和 **更新** 变体。例如，**创建** 变体可能需要所有字段，而 **更新** 变体可能使所有字段可选。

Nest 提供了 `PartialType()` 实用函数，使这项任务更简单，减少样板代码。

`PartialType()` 函数返回一个类型（类），将输入类型的所有属性设置为可选。例如，假设我们有一个 **创建** 类型如下：

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

默认情况下，所有这些字段都是必需的。要创建一个具有相同字段但每个字段都是可选的类型，使用 `PartialType()` 传递类引用（`CreateCatDto`）作为参数：

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

> 信息 **提示** `PartialType()` 函数从 `@nestjs/swagger` 包导入。

#### 选择（Pick）

`PickType()` 函数通过从输入类型中选择一组属性来构建新类型（类）。例如，假设我们从以下类型开始：

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

我们可以使用 `PickType()` 实用函数从这个类中选择一组属性：

```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

> 信息 **提示** `PickType()` 函数从 `@nestjs/swagger` 包导入。

#### 省略（Omit）

`OmitType()` 函数通过从输入类型中选择所有属性，然后移除特定一组键来构建类型。例如，假设我们从以下类型开始：

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

我们可以生成一个派生类型，该类型具有除了 `name` 之外的所有属性，如下所示。在这个构造中，`OmitType` 的第二个参数是属性名数组。

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

> 信息 **提示** `OmitType()` 函数从 `@nestjs/swagger` 包导入。

#### 交集（Intersection）

`IntersectionType()` 函数将两个类型合并为一个新类型（类）。例如，假设我们从两个类型开始：

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  breed: string;
}

export class AdditionalCatInfo {
  @ApiProperty()
  color: string;
}
```

我们可以生成一个新类型，该类型结合了两种类型中的所有属性。

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

> 信息 **提示** `IntersectionType()` 函数从 `@nestjs/swagger` 包导入。

#### 组合

类型映射实用函数是可组合的。例如，以下将产生一个类型（类），该类型具有 `CreateCatDto` 类型的所有属性，除了 `name`，并且这些属性将被设置为可选：

```typescript
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```