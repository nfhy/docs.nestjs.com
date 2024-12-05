### CLI命令参考

#### nest new

创建一个新的（标准模式）Nest项目。

```bash
$ nest new <name> [options]
$ nest n <name> [options]
```

##### 描述

创建并初始化一个新的Nest项目。提示选择包管理器。

- 创建一个名为`<name>`的文件夹
- 在文件夹中填充配置文件
- 为源代码（`/src`）和端到端测试（`/test`）创建子文件夹
- 在子文件夹中填充应用程序组件和测试的默认文件

##### 参数

| 参数   | 描述                     |
| ------ | ------------------------ |
| `<name>` | 新项目的名字 |

##### 选项

| 选项                                | 描述                                                                                         |
| ----------------------------------- | -------------------------------------------------------------------------------------------- |
| `--dry-run`                         | 报告将要进行的更改，但不改变文件系统。<br/>别名：`-d`                                           |
| `--skip-git`                        | 跳过git仓库初始化。<br/>别名：`-g`                                                          |
| `--skip-install`                    | 跳过包安装。<br/>别名：`-s`                                                                 |
| `--package-manager [package-manager]` | 指定包管理器。使用`npm`、`yarn`或`pnpm`。包管理器必须全局安装。<br/>别名：`-p`                  |
| `--language [language]`             | 指定编程语言（`TS`或`JS`）。<br/>别名：`-l`                                                |
| `--collection [collectionName]`     | 指定schematics集合。使用包含schematics的已安装npm包的包名。<br/>别名：`-c`                      |
| `--strict`                         | 启动项目时启用以下TypeScript编译器标志：`strictNullChecks`、`noImplicitAny`、`strictBindCallApply`、`forceConsistentCasingInFileNames`、`noFallthroughCasesInSwitch` |

#### nest generate

基于schematics生成和/或修改文件

```bash
$ nest generate <schematic> <name> [options]
$ nest g <schematic> <name> [options]
```

##### 参数

| 参数         | 描述                                                                                              |
| ------------ | -------------------------------------------------------------------------------------------------- |
| `<schematic>` | 要生成的`schematic`或`collection:schematic`。见下表了解可用的schematics。 |
| `<name>`      | 生成的组件名称。                                                                                      |

##### Schematics

| 名称          | 别名 | 描述                                                                                                            |
| ------------- | ---- | ---------------------------------------------------------------------------------------------------------- |
| `app`         |      | 在monorepo中生成一个新的应用程序（如果它是标准结构，则转换为monorepo）。                                    |
| `library`     | `lib` | 在monorepo中生成一个新的库（如果它是标准结构，则转换为monorepo）。                                       |
| `class`       | `cl`  | 生成一个新类。                                                                                              |
| `controller`  | `co`  | 生成一个控制器声明。                                                                                      |
| `decorator`   | `d`   | 生成一个自定义装饰器。                                                                                      |
| `filter`      | `f`   | 生成一个过滤器声明。                                                                                       |
| `gateway`     | `ga`  | 生成一个网关声明。                                                                                       |
| `guard`       | `gu`  | 生成一个守卫声明。                                                                                       |
| `interface`   | `itf` | 生成一个接口。                                                                                           |
| `interceptor` | `itc` | 生成一个拦截器声明。                                                                                      |
| `middleware`  | `mi`  | 生成一个中间件声明。                                                                                      |
| `module`      | `mo`  | 生成一个模块声明。                                                                                       |
| `pipe`        | `pi`  | 生成一个管道声明。                                                                                       |
| `provider`    | `pr`  | 生成一个提供者声明。                                                                                      |
| `resolver`    | `r`   | 生成一个解析器声明。                                                                                      |
| `resource`    | `res` | 生成一个新的CRUD资源。更多详情见[CRUD（资源）生成器](/recipes/crud-generator)。（仅限TS）                      |
| `service`     | `s`   | 生成一个服务声明。                                                                                       |

##### 选项

| 选项                          | 描述                                                                                                     |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `--dry-run`                     | 报告将要进行的更改，但不改变文件系统。<br/>别名：`-d`                                                        |
| `--project [project]`           | 元素应该添加到的项目。<br/>别名：`-p`                                                                      |
| `--flat`                        | 不为元素生成文件夹。                                                                                      |
| `--collection [collectionName]` | 指定schematics集合。使用包含schematics的已安装npm包的包名。<br/>别名：`-c`                                   |
| `--spec`                        | 强制生成spec文件（默认）                                                                                 |
| `--no-spec`                     | 禁用spec文件生成                                                                                       |

#### nest build

编译应用程序或工作区到输出文件夹。

同时，`build`命令还负责：

- 通过`tsconfig-paths`映射路径（如果使用路径别名）
- 注解DTOs与OpenAPI装饰器（如果启用了`@nestjs/swagger` CLI插件）
- 注解DTOs与GraphQL装饰器（如果启用了`@nestjs/graphql` CLI插件）

```bash
$ nest build <name> [options]
```

##### 参数

| 参数   | 描述                       |
| ------ | -------------------------- |
| `<name>` | 要构建的项目的名字。 |

##### 选项

| 选项             | 描述                                                                                                                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--path [path]`    | `tsconfig`文件的路径。<br/>别名 `-p`                                                                                                                                                    |
| `--config [path]`  | `nest-cli`配置文件的路径。<br/>别名 `-c`                                                                                                                                     |
| `--watch`          | 在监视模式下运行（实时重载）。<br / >如果您使用`tsc`进行编译，可以在`manualRestart`选项设置为`true`时输入`rs`重启应用程序。<br/>别名 `-w` |
| `--builder [name]` | 指定用于编译的构建器（`tsc`、`swc`或`webpack`）。<br/>别名 `-b`                                                                                                   |
| `--webpack`        | 使用webpack进行编译（已弃用：请使用`--builder webpack`代替）。                                                                                                                 |
| `--webpackPath`    | webpack配置的路径。                                                                                                                                                             |
| `--tsc`            | 强制使用`tsc`进行编译。                                                                                                                                                           |

#### nest start

编译并运行应用程序（或工作区中的默认项目）。

```bash
$ nest start <name> [options]
```

##### 参数

| 参数   | 描述                     |
| ------ | ------------------------ |
| `<name>` | 要运行的项目的名字。 |

##### 选项

| 选项                  | 描述                                                                                                            |
| --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `--path [path]`         | `tsconfig`文件的路径。<br/>别名 `-p`                                                                            |
| `--config [path]`       | `nest-cli`配置文件的路径。<br/>别名 `-c`                                                                   |
| `--watch`               | 在监视模式下运行（实时重载）<br/>别名 `-w`                                                                      |
| `--builder [name]`      | 指定用于编译的构建器（`tsc`、`swc`或`webpack`）。<br/>别名 `-b`                             |
| `--preserveWatchOutput` | 在监视模式下保持过时的控制台输出而不是清除屏幕。（仅`tsc`监视模式）                   |
| `--watchAssets`         | 在监视模式下运行（实时重载），监视非TS文件（资产）。更多详情见[资产](cli/monorepo#assets)。 |
| `--debug [hostport]`    | 在调试模式下运行（带有--inspect标志）<br/>别名 `-d`                                                              |
| `--webpack`             | 使用webpack进行编译。（已弃用：请使用`--builder webpack`代替）                                           |
| `--webpackPath`         | webpack配置的路径。                                                                                       |
| `--tsc`                 | 强制使用`tsc`进行编译。                                                                                     |
| `--exec [binary]`       | 要运行的二进制文件（默认：`node`）。<br/>别名 `-e`                                                                     |
| `-- [key=value]`        | 可以通过`process.argv`引用的命令行参数。                                                    |

#### nest add

导入一个被打包为**nest库**的库，运行其安装schematics。

```bash
$ nest add <name> [options]
```

##### 参数

| 参数   | 描述                        |
| ------ | --------------------------- |
| `<name>` | 要导入的库的名字。 |

#### nest info

显示有关安装的nest包和其他有用的系统信息。例如：

```bash
$ nest info
```

```bash
 _   _             _      ___  _____  _____  _     _____  
| \ | |           | |    |_  |/  ___|/  __ \| |   |_   _|
|  \| |  ___  ___ | |_     | |\\ `--. | /  \/| |     | |  
| . ` | / _ \/ __|| __|    | | `--. \| |    | |     | |  
| |\   ||  __/\__ \| |_ /\\__/ //\__/ /| \__/\| |_____| |_ 
\_| \_/ \___||___/ \__|\____/ \____/  \____/\_____/\___/

[System Information]
OS Version : macOS High Sierra
NodeJS Version : v16.18.0
[Nest Information]
microservices version : 10.0.0
websockets version : 10.0.0
testing version : 10.0.0
common version : 10.0.0
core version : 10.0.0
```