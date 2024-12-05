### 跨源资源共享（CORS）

跨源资源共享（CORS）是一种允许从另一个域请求资源的机制。在底层，Nest根据底层平台使用Express的[cors](https://github.com/expressjs/cors)或Fastify的[@fastify/cors](https://github.com/fastify/fastify-cors)包。这些包提供了各种选项，你可以根据你的需求进行自定义。

#### 开始使用

要启用CORS，请在Nest应用程序对象上调用`enableCors()`方法。

```typescript
const app = await NestFactory.create(AppModule);
app.enableCors();
await app.listen(process.env.PORT ?? 3000);
```

`enableCors()`方法接受一个可选的配置对象参数。这个对象的可用属性在官方[CORS](https://github.com/expressjs/cors#configuration-options)文档中有描述。另一种方式是传递一个[回调函数](https://github.com/expressjs/cors#configuring-cors-asynchronously)，它允许你根据请求（即时）异步定义配置对象。

或者，可以通过`create()`方法的选项对象启用CORS。将`cors`属性设置为`true`以使用默认设置启用CORS。或者，将一个[CORS配置对象](https://github.com/expressjs/cors#configuration-options)或[回调函数](https://github.com/expressjs/cors#configuring-cors-asynchronously)作为`cors`属性值传递以自定义其行为。

```typescript
const app = await NestFactory.create(AppModule, { cors: true });
await app.listen(process.env.PORT ?? 3000);
```