### 文档说明

**Compodoc** 是一个用于 Angular 应用程序的文档工具。由于 Nest 和 Angular 拥有相似的项目和代码结构，**Compodoc** 同样适用于 Nest 应用程序。

#### 设置

在现有的 Nest 项目中设置 Compodoc 非常简单。首先，通过在操作系统的终端中执行以下命令来添加开发依赖：

```bash
$ npm i -D @compodoc/compodoc
```

#### 生成

使用以下命令生成项目文档（需要 npm 6 以支持 `npx`）。更多选项请查看[官方文档](https://compodoc.app/guides/usage.html)。

```bash
$ npx @compodoc/compodoc -p tsconfig.json -s
```

打开浏览器，导航至 [http://localhost:8080](http://localhost:8080)。您应该可以看到一个初始的 Nest CLI 项目：

<figure><img src="/assets/documentation-compodoc-1.jpg" /></figure>
<figure><img src="/assets/documentation-compodoc-2.jpg" /></figure>

#### 贡献

您可以在[这里](https://github.com/compodoc/compodoc)参与并贡献 Compodoc 项目。