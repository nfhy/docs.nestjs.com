### 生成 SDL

> 警告 **警告** 本章节仅适用于代码优先方法。

要手动生成 GraphQL SDL 架构（即，不运行应用程序、不连接数据库、不连接解析器等），请使用 `GraphQLSchemaBuilderModule`。

```typescript
async function generateSchema() {
  const app = await NestFactory.create(GraphQLSchemaBuilderModule);
  await app.init();

  const gqlSchemaFactory = app.get(GraphQLSchemaFactory);
  const schema = await gqlSchemaFactory.create([RecipesResolver]);
  console.log(printSchema(schema));
}
```

> 提示 **提示** `GraphQLSchemaBuilderModule` 和 `GraphQLSchemaFactory` 是从 `@nestjs/graphql` 包导入的。`printSchema` 函数是从 `graphql` 包导入的。

#### 使用方法

`gqlSchemaFactory.create()` 方法接受一个解析器类引用数组。例如：

```typescript
const schema = await gqlSchemaFactory.create([
  RecipesResolver,
  AuthorsResolver,
  PostsResolvers,
]);
```

它还接受一个可选的第二个参数，包含标量类的数组：

```typescript
const schema = await gqlSchemaFactory.create(
  [RecipesResolver, AuthorsResolver, PostsResolvers],
  [DurationScalar, DateScalar],
);
```

最后，您可以传递一个选项对象：

```typescript
const schema = await gqlSchemaFactory.create([RecipesResolver], {
  skipCheck: true,
  orphanedTypes: [],
});
```

- `skipCheck`: 忽略架构验证；布尔值，默认为 `false`
- `orphanedTypes`: 未被明确引用的类列表（不是对象图的一部分）将被生成。通常，如果一个类被声明但没有在图中被引用，它将被省略。属性值是一个类引用数组。