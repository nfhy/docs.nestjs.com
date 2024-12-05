### 热重载

对您的应用程序启动过程影响最大的是 **TypeScript编译**。幸运的是，通过[webpack](https://github.com/webpack/webpack)的HMR（热模块替换），我们不需要每次更改时都重新编译整个项目。这大大减少了实例化应用程序所需的时间，并使迭代开发变得更加容易。

> **警告** 注意，`webpack`不会自动将您的资产（例如`graphql`文件）复制到`dist`文件夹。同样，`webpack`与glob静态路径不兼容（例如`TypeOrmModule`中的`entities`属性）。

### 使用CLI

如果您使用的是[Nest CLI](https://docs.nestjs.com/cli/overview)，配置过程非常简单。CLI包装了`webpack`，允许使用`HotModuleReplacementPlugin`。

#### 安装

首先安装所需的包：

```bash
$ npm i --save-dev webpack-node-externals run-script-webpack-plugin webpack
```

> **提示** 如果您使用的是**Yarn Berry**（而不是经典Yarn），请安装`webpack-pnp-externals`包而不是`webpack-node-externals`。

#### 配置

安装完成后，在应用程序的根目录下创建一个`webpack-hmr.config.js`文件。

```typescript
const nodeExternals = require('webpack-node-externals');
const { RunScriptWebpackPlugin } = require('run-script-webpack-plugin');

module.exports = function (options, webpack) {
  return {
    ...options,
    entry: ['webpack/hot/poll?100', options.entry],
    externals: [
      nodeExternals({
        allowlist: ['webpack/hot/poll?100'],
      }),
    ],
    plugins: [
      ...options.plugins,
      new webpack.HotModuleReplacementPlugin(),
      new webpack.WatchIgnorePlugin({
        paths: [/\.js$/, /\.d\.ts$/],
      }),
      new RunScriptWebpackPlugin({ name: options.output.filename, autoRestart: false }),
    ],
  };
};
```

> **提示** 使用**Yarn Berry**（而不是经典Yarn）时，在`externals`配置属性中不使用`nodeExternals`，而是使用`webpack-pnp-externals`包中的`WebpackPnpExternals`：`WebpackPnpExternals({{ '{' }} exclude: ['webpack/hot/poll?100'] {{ '}' }})`。

这个函数接受包含默认webpack配置的对象作为第一个参数，并将Nest CLI使用的底层`webpack`包的引用作为第二个参数。同时，它返回一个修改过的webpack配置，其中包含了`HotModuleReplacementPlugin`、`WatchIgnorePlugin`和`RunScriptWebpackPlugin`插件。

#### 热模块替换

要启用**HMR**，请打开应用程序的入口文件（`main.ts`）并添加以下与webpack相关的指令：

```typescript
declare const module: any;

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);

  if (module.hot) {
    module.hot.accept();
    module.hot.dispose(() => app.close());
  }
}
bootstrap();
```

为了简化执行过程，请在您的`package.json`文件中添加一个脚本。

```json
"start:dev": "nest build --webpack --webpackPath webpack-hmr.config.js --watch"
```

现在只需打开命令行并运行以下命令：

```bash
$ npm run start:dev
```

### 不使用CLI

如果您不使用[Nest CLI](https://docs.nestjs.com/cli/overview)，配置将稍微复杂一些（需要更多的手动步骤）。

#### 安装

首先安装所需的包：

```bash
$ npm i --save-dev webpack webpack-cli webpack-node-externals ts-loader run-script-webpack-plugin
```

> **提示** 如果您使用的是**Yarn Berry**（而不是经典Yarn），请安装`webpack-pnp-externals`包而不是`webpack-node-externals`。

#### 配置

安装完成后，在应用程序的根目录下创建一个`webpack.config.js`文件。

```typescript
const webpack = require('webpack');
const path = require('path');
const nodeExternals = require('webpack-node-externals');
const { RunScriptWebpackPlugin } = require('run-script-webpack-plugin');

module.exports = {
  entry: ['webpack/hot/poll?100', './src/main.ts'],
  target: 'node',
  externals: [
    nodeExternals({
      allowlist: ['webpack/hot/poll?100'],
    }),
  ],
  module: {
    rules: [
      {
        test: /.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
    ],
  },
  mode: 'development',
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
  plugins: [new webpack.HotModuleReplacementPlugin(), new RunScriptWebpackPlugin({ name: 'server.js', autoRestart: false })],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: 'server.js',
  },
};
```

> **提示** 使用**Yarn Berry**（而不是经典Yarn）时，在`externals`配置属性中不使用`nodeExternals`，而是使用`webpack-pnp-externals`包中的`WebpackPnpExternals`：`WebpackPnpExternals({{ '{' }} exclude: ['webpack/hot/poll?100'] {{ '}' }})`。

这个配置告诉webpack一些关于您的应用程序的基本事项：入口文件的位置，应该使用哪个目录来保存**编译**后的文件，以及我们想要使用哪种加载器来编译源文件。通常，即使您不完全理解所有选项，也应该能够直接使用这个文件。

#### 热模块替换

要启用**HMR**，请打开应用程序的入口文件（`main.ts`）并添加以下与webpack相关的指令：

```typescript
declare const module: any;

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);

  if (module.hot) {
    module.hot.accept();
    module.hot.dispose(() => app.close());
  }
}
bootstrap();
```

为了简化执行过程，请在您的`package.json`文件中添加一个脚本。

```json
"start:dev": "webpack --config webpack.config.js --watch"
```

现在只需打开命令行并运行以下命令：

```bash
$ npm run start:dev
```

#### 示例

一个工作示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/08-webpack)找到。