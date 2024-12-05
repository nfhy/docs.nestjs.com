### 共享模型

> 警告 **警告** 本章节仅适用于代码优先方法。

使用Typescript作为项目后端的一个最大优势是能够在Typescript前端应用程序中重用相同的模型，通过使用一个公共的Typescript包。

但是存在一个问题：使用代码优先方法创建的模型被大量与GraphQL相关的装饰器所装饰。这些装饰器在前端是无关紧要的，会对性能产生负面影响。

#### 使用模型遮罩

为了解决这个问题，NestJS提供了一个“遮罩”，它允许你通过使用`webpack`（或类似的）配置，将原始装饰器替换为惰性代码。

要使用这个遮罩，配置一个别名在`@nestjs/graphql`包和遮罩之间。

例如，对于webpack，可以这样解决：

```typescript
resolve: { // 参见：https://webpack.js.org/configuration/resolve/
  alias: {
      "@nestjs/graphql": path.resolve(__dirname, "../node_modules/@nestjs/graphql/dist/extra/graphql-model-shim")
  }
}
```

> 提示 **提示** [TypeORM](/techniques/database) 包有一个类似的遮罩，可以在这里找到 [这里](https://github.com/typeorm/typeorm/blob/master/extra/typeorm-model-shim.js)。