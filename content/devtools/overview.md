### 概览

> 提示：本章涵盖了Nest Devtools与Nest框架的集成。如果您正在寻找Devtools应用程序，请访问[Devtools](https://devtools.nestjs.com)网站。

要开始调试您的本地应用程序，请打开`main.ts`文件，并确保在应用程序选项对象中将`snapshot`属性设置为`true`，如下所示：

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    snapshot: true,
  });
  await app.listen(process.env.PORT ?? 3000);
}
```

这将指示框架收集必要的元数据，以便Nest Devtools可以可视化您的应用程序图。

接下来，让我们安装所需的依赖项：

```bash
$ npm i @nestjs/devtools-integration
```

> 警告：如果您的应用程序中使用了`@nestjs/graphql`包，请确保安装最新版本（`npm i @nestjs/graphql@11`）。

有了这个依赖项，让我们打开`app.module.ts`文件，并导入我们刚刚安装的`DevtoolsModule`：

```typescript
@Module({
  imports: [
    DevtoolsModule.register({
      http: process.env.NODE_ENV !== 'production',
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

> 警告：我们在这里检查`NODE_ENV`环境变量的原因是您永远不应该在生产环境中使用这个模块！

一旦导入了`DevtoolsModule`并且您的应用程序正在运行（`npm run start:dev`），您应该能够导航到[Devtools](https://devtools.nestjs.com) URL并看到内省图。

<figure><img src="/assets/devtools/modules-graph.png" /></figure>

> 提示：正如您在上图中看到的，每个模块都连接到`InternalCoreModule`。`InternalCoreModule`是一个全局模块，它总是被导入到根模块中。由于它被注册为全局节点，Nest自动在所有模块和`InternalCoreModule`节点之间创建边。现在，如果您想从图中隐藏全局模块，您可以使用“**隐藏全局模块**”复选框（在侧边栏中）。

正如我们所看到的，`DevtoolsModule`使您的应用程序暴露了一个额外的HTTP服务器（在端口8000上），Devtools应用程序将使用它来内省您的应用程序。

为了再次确认一切按预期工作，请将图视图更改为“Classes”。您应该看到以下屏幕：

<figure><img src="/assets/devtools/classes-graph.png" /></figure>

要专注于特定节点，单击矩形，图将显示一个弹出窗口，带有**“Focus”**按钮。您还可以使用搜索栏（位于侧边栏）来查找特定节点。

> 提示：如果您单击**Inspect**按钮，应用程序将带您进入`/debug`页面，并选择该特定节点。

<figure><img src="/assets/devtools/node-popup.png" /></figure>

> 提示：要将图导出为图片，请单击图右上角的**Export as PNG**按钮。

使用位于侧边栏（左侧）的表单控件，您可以控制边的接近程度，例如，可视化特定应用程序子树：

<figure><img src="/assets/devtools/subtree-view.png" /></figure>

这在您有**新开发人员**加入团队时特别有用，您想向他们展示您的应用程序是如何构建的。您还可以使用此功能来可视化特定模块（例如`TasksModule`）及其所有依赖项，这在您将大型应用程序分解为较小模块（例如，单独的微服务）时非常有用。

您可以观看此视频以查看**Graph Explorer**功能的实际效果：

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/bW8V-ssfnvM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### 调查“无法解析依赖”错误

> 注意：此功能支持`@nestjs/core` >= `v9.3.10`。

您可能遇到的最常见的错误消息是关于Nest无法解析提供者的依赖项。使用Nest Devtools，您可以轻松识别问题并了解如何解决它。

首先，打开`main.ts`文件并更新`bootstrap()`调用，如下所示：

```typescript
bootstrap().catch((err) => {
  fs.writeFileSync('graph.json', PartialGraphHost.toString() ?? '');
  process.exit(1);
});
```

同时，确保将`abortOnError`设置为`false`：

```typescript
const app = await NestFactory.create(AppModule, {
  snapshot: true,
  abortOnError: false, // <--- THIS
});
```

现在，每次您的应用程序因**“无法解析依赖”**错误而无法启动时，您都会在根目录中找到`graph.json`文件（表示部分图）。然后，您可以将此文件拖放（确保将当前模式从“Interactive”切换到“Preview”）到Devtools：

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

上传成功后，您应该看到以下图和对话框：

<figure><img src="/assets/devtools/partial-graph-modules-view.png" /></figure>

如您所见，突出显示的`TasksModule`是我们应当查看的模块。此外，在对话框中，您已经可以看到一些关于如何解决此问题的说明。

如果我们切换到“Classes”视图，我们将看到：

<figure><img src="/assets/devtools/partial-graph-classes-view.png" /></figure>

此图说明了我们想要注入到`TasksService`中的`DiagnosticsService`在`TasksModule`模块的上下文中未找到，我们可能只需要将`DiagnosticsModule`导入到`TasksModule`模块中就可以解决这个问题！

#### 路由浏览器

当您导航到**路由浏览器**页面时，您应该看到所有注册的入口点：

<figure><img src="/assets/devtools/routes.png" /></figure>

> 提示：此页面不仅显示HTTP路由，还显示所有其他入口点（例如WebSockets、gRPC、GraphQL解析器等）。

入口点按其主机控制器分组。您还可以使用搜索栏来查找特定的入口点。

如果您单击特定的入口点，将显示**流程图**。此图显示了入口点的执行流程（例如，绑定到此路由的守卫、拦截器、管道等）。当您想要了解特定路由的请求/响应周期看起来如何，或者当您遇到特定守卫/拦截器/管道未被执行的问题时，这特别有用。

#### 沙箱

要执行即时JavaScript代码并与您的应用程序实时交互，请导航到**沙箱**页面：

<figure><img src="/assets/devtools/sandbox.png" /></figure>

游乐场可用于实时测试和调试API端点，允许开发人员快速识别和修复问题，而无需使用例如HTTP客户端。我们还可以绕过身份验证层，因此我们不再需要登录的额外步骤，甚至不需要用于测试的特殊用户帐户。对于事件驱动的应用程序，我们还可以从游乐场直接触发事件，并查看应用程序如何响应它们。

任何被记录的内容都会流式传输到游乐场的控制台，因此我们可以轻松地了解发生了什么。

只需执行代码**即时**并立即查看结果，无需重建应用程序并重新启动服务器。

<figure><img src="/assets/devtools/sandbox-table.png" /></figure>

> 提示：要美观地显示对象数组，请使用`console.table()`（或仅`table()`）函数。

您可以观看此视频以查看**交互式游乐场**功能的实际效果：

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/liSxEN_VXKM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### 引导性能分析器

要查看所有类节点（控制器、提供者、增强器等）及其相应的实例化时间的列表，请导航到**引导性能**页面：

<figure><img src="/assets/devtools/bootstrap-performance.png" /></figure>

当您想要识别应用程序引导过程中的最慢部分时（例如，当您想要优化应用程序的启动时间，这对于例如无服务器环境至关重要时），此页面特别有用。

#### 审计

要查看应用程序在分析您的序列化图时生成的自动审计 - 错误/警告/提示，请导航到**审计**页面：

<figure><img src="/assets/devtools/audit.png" /></figure>

> 提示：上面的截图没有显示所有可用的审计规则。

当您想要识别应用程序中的潜在问题时，此页面非常有用。

#### 预览静态文件

要将序列化图保存到文件，请使用以下代码：

```typescript
await app.listen(process.env.PORT ?? 3000); // OR await app.init()
fs.writeFileSync('./graph.json', app.get(SerializedGraph).toString());
```

> 提示：`SerializedGraph`是从`@nestjs/core`包中导出的。

然后您可以拖放/上传此文件：

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

这在您想要与其他人（例如，同事）共享您的图，或当您想要离线分析它时非常有用。