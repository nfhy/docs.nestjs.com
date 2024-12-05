### 部署

当你准备好将你的 NestJS 应用程序部署到生产环境时，有一些关键步骤可以帮助你确保应用程序尽可能高效地运行。在本指南中，我们将探讨部署你的 NestJS 应用程序成功的必备技巧和最佳实践。

#### 先决条件

在部署你的 NestJS 应用程序之前，请确保你已经：

- 有一个准备好部署的工作 NestJS 应用程序。
- 可以访问部署平台或服务器，以便你可以托管你的应用程序。
- 为你的应用程序设置所有必要的环境变量。
- 像数据库这样的任何所需服务都已设置并准备就绪。
- 在你的部署平台上至少安装了 LTS 版本的 Node.js。

> 信息 **提示** 如果你正在寻找一个云平台来部署你的 NestJS 应用程序，请查看 [Mau](https://mau.nestjs.com/ 'Deploy Nest')，这是我们在 AWS 上部署 NestJS 应用程序的官方平台。有了 Mau，部署你的 NestJS 应用程序就像点击几个按钮并运行一个命令一样简单：
> 
> ```
> $ npm install -g @nestjs/mau
> $ mau deploy
> ```
> 
> 部署完成后，你的 NestJS 应用程序将在短时间内在 AWS 上运行！

#### 构建你的应用程序

要构建你的 NestJS 应用程序，你需要将你的 TypeScript 代码编译成 JavaScript。这个过程会生成一个包含编译文件的 `dist` 目录。你可以通过运行以下命令来构建你的应用程序：

```bash
$ npm run build
```

这个命令通常在后台运行 `nest build` 命令，这基本上是一个围绕 TypeScript 编译器的包装器，具有一些额外的功能（例如复制资产等）。如果你有一个自定义的构建脚本，你可以直接运行它。另外，对于 NestJS CLI 单仓库项目，请确保将要构建的项目名称作为参数传递（`npm run build my-app`）。

成功编译后，你应该在项目根目录中看到一个 `dist` 目录，其中包含编译后的文件，入口点是 `main.js`。如果你有任何位于项目根目录的 `.ts` 文件（并且你的 `tsconfig.json` 配置为编译它们），它们也会被复制到 `dist` 目录中，稍微修改目录结构（不是 `dist/main.js`，而是 `dist/src/main.js`，所以在配置服务器时请注意这一点）。

#### 生产环境

你的生产环境是你应用程序对外部用户可访问的地方。这可能是像 [AWS](https://aws.amazon.com/)（带有 EC2、ECS 等）这样的基于云的平台，[Azure](https://azure.microsoft.com/) 或 [Google Cloud](https://cloud.google.com/)，甚至是你自己管理的专用服务器，比如 [Hetzner](https://www.hetzner.com/)。

为了简化部署过程并避免手动设置，你可以使用像 [Mau](https://mau.nestjs.com/ 'Deploy Nest') 这样的服务，这是我们在 AWS 上部署 NestJS 应用程序的官方平台。更多详情，请查看[这一节](todo)。

使用 **基于云的平台** 或像 [Mau](https://mau.nestjs.com/ 'Deploy Nest') 这样的服务的一些优点包括：

- 可扩展性：随着用户基础的增长，轻松扩展你的应用程序。
- 安全性：从内置的安全功能和合规认证中受益。
- 监控：实时监控你的应用程序的性能和健康状况。
- 可靠性：确保你的应用程序始终可用，具有高正常运行时间保证。

另一方面，基于云的平台通常比自托管更昂贵，并且你可能对底层基础设施的控制较少。如果你正在寻找更具成本效益的解决方案，并且有技术专长来自己管理服务器，简单的 VPS 可能是一个不错的选择，但请记住，你需要手动处理服务器维护、安全和备份等任务。

#### NODE_ENV=production

虽然在 Node.js 和 NestJS 中，开发环境和生产环境之间技术上没有区别，但在生产环境中运行应用程序时，将 `NODE_ENV` 环境变量设置为 `production` 是一个好习惯，因为生态系统中的一些库可能会根据这个变量有不同的行为（例如，启用或禁用调试输出等）。

你可以在启动应用程序时设置 `NODE_ENV` 环境变量，如下所示：

```bash
$ NODE_ENV=production node dist/main.js
```

或者，你可以在云提供商/Mau仪表板中设置它。

#### 运行你的应用程序

在生产环境中运行你的 NestJS 应用程序，只需使用以下命令：

```bash
$ node dist/main.js # 根据你的入口点位置进行调整
```

这个命令启动你的应用程序，它将在指定的端口（默认通常是 `3000`）上监听。确保这与你在应用程序中配置的端口相匹配。

或者，你可以使用 `nest start` 命令。这个命令是 `node dist/main.js` 的包装器，但它有一个关键的区别：它在启动应用程序之前自动运行 `nest build`，所以你不需要手动执行 `npm run build`。

#### 健康检查

健康检查对于监控生产环境中的 NestJS 应用程序的健康状况至关重要。通过设置健康检查端点，你可以定期验证你的应用程序是否按预期运行，并在问题变得严重之前做出响应。

在 NestJS 中，你可以使用 **@nestjs/terminus** 包轻松实现健康检查，该包提供了一个强大的工具，用于添加健康检查，包括数据库连接、外部服务和自定义检查。

查看[这个指南](/recipes/terminus)，了解如何在你的 NestJS 应用程序中实现健康检查，并确保你的应用程序始终受到监控和响应。

#### 日志记录

日志记录对于任何生产就绪的应用程序都是必不可少的。它有助于跟踪错误、监控行为和解决问题。在 NestJS 中，你可以使用内置的日志记录器轻松管理日志记录，或者如果你需要更高级的功能，可以选择外部库。

日志记录的最佳实践：

- 记录错误，而不是异常：专注于记录详细的错误消息，以加快调试和问题解决的速度。
- 避免敏感数据：永远不要记录敏感信息，如密码或令牌，以保护安全。
- 使用相关 ID：在分布式系统中，包括唯一的标识符（如相关 ID）在你的日志中，以追踪跨不同服务的请求。
- 使用日志级别：按严重程度对日志进行分类（例如 `info`、`warn`、`error`），并在生产中禁用调试或详细日志以减少噪音。

> 信息 **提示** 如果你使用 [AWS](https://aws.amazon.com/)（带有 [Mau](https://mau.nestjs.com/ 'Deploy Nest') 或直接使用），考虑使用 JSON 日志记录，以便于解析和分析你的日志。

对于分布式应用程序，使用像 ElasticSearch、Loggly 或 Datadog 这样的集中式日志服务非常有用。这些工具提供了日志聚合、搜索和可视化等强大功能，使监控和分析应用程序的性能和行为变得更容易。

#### 扩展或扩展

有效地扩展你的 NestJS 应用程序对于处理增加的流量和确保最佳性能至关重要。有两种主要的扩展策略：**垂直扩展** 和 **水平扩展**。了解这些方法将帮助你设计你的应用程序以有效地管理负载。

**垂直扩展**，通常称为“扩展”，涉及增加单个服务器的资源以提高其性能。这可能意味着为你现有的机器添加更多的 CPU、RAM 或存储。以下是一些需要考虑的关键点：

- 简单性：垂直扩展通常更简单，因为你只需要升级你现有的服务器，而不需要管理多个实例。
- 限制：单个机器的扩展有物理限制。一旦达到最大容量，你可能需要考虑其他选项。
- 成本效益：对于流量适中的应用程序，垂直扩展可以节省成本，因为它减少了对额外基础设施的需求。

示例：如果你的 NestJS 应用程序托管在虚拟机上，你注意到在高峰时段运行缓慢，你可以将你的 VM 升级到具有更多资源的更大实例。要升级你的 VM，只需导航到你当前提供商的仪表板并选择一个更大的实例类型。

**水平扩展**，或“扩展”，涉及添加更多服务器或实例来分散负载。这种策略在云环境中广泛使用，对于预期高流量的应用程序至关重要。以下是好处和考虑因素：

- 增加容量：通过添加更多应用程序实例，你可以处理更多的并发用户，而不会降低性能。
- 冗余：水平扩展提供冗余，因为一个服务器的故障不会使你的整个应用程序崩溃。流量可以在剩余的服务器之间重新分配。
- 负载均衡：要有效地管理多个实例，使用负载均衡器（如 Nginx 或 AWS 弹性负载均衡）在服务器之间均匀分配传入流量。

示例：对于一个经历高流量的 NestJS 应用程序，你可以在云环境中部署多个应用程序实例，并使用负载均衡器来路由请求，确保没有单个实例成为瓶颈。

这个过程在像 [Docker](https://www.docker.com/) 这样的容器化技术和像 [Kubernetes](https://kubernetes.io/) 这样的容器编排平台的帮助下变得简单。此外，你可以利用云特定的负载均衡器，如 [AWS 弹性负载均衡](https://aws.amazon.com/elasticloadbalancing/) 或 [Azure 负载均衡器](https://azure.microsoft.com/en-us/services/load-balancer/) 来跨应用程序实例分发流量。

> 信息 **提示** [Mau](https://mau.nestjs.com/ 'Deploy Nest') 在 AWS 上提供内置的水平扩展支持，允许你轻松部署多个 NestJS 应用程序实例，并只需几次点击即可管理它们。

#### 其他一些提示

在部署你的 NestJS 应用程序时，还有一些其他的提示需要记住：

- **安全**：确保你的应用程序安全并受到保护，免受 SQL 注入、XSS 等常见威胁的侵害。查看“安全”类别以获取更多详细信息。
- **监控**：使用像 [Prometheus](https://prometheus.io/) 或 [New Relic](https://newrelic.com/) 这样的监控工具来跟踪你的应用程序的性能和健康状况。如果你使用云提供商/Mau，他们可能提供内置的监控服务（如 [AWS CloudWatch](https://aws.amazon.com/cloudwatch/) 等）。
- **不要硬编码环境变量**：避免在代码中硬编码敏感信息，如 API 密钥、密码或令牌。使用环境变量或秘密管理器来安全地存储和访问这些值。
- **备份**：定期备份你的数据，以防在发生事故时丢失数据。
- **自动化部署**：使用 CI/CD 管道来自动化你的部署过程，并确保跨环境的一致性。
- **速率限制**：实施速率限制以防止滥用并保护你的应用程序免受 DDoS 攻击。查看 [速率限制章节](/security/rate-limiting) 以获取更多详细信息，或使用像 [AWS WAF](https://aws.amazon.com/waf/) 这样的服务进行高级保护。

#### 将你的应用程序容器化

[Docker](https://www.docker.com/) 是一个使用容器化的平台，允许开发人员将应用程序及其依赖项打包到一个名为容器的标准单元中。容器轻量级、便携且隔离，使它们成为在各种环境中部署应用程序的理想选择，从本地开发到生产。

容器化你的 NestJS 应用程序的好处：

- 一致性：Docker 确保你的应用程序在任何机器上以相同的方式运行，消除了“在我的机器上可以工作”的问题。
- 隔离：每个容器在其隔离的环境中运行，防止依赖项之间的冲突。
- 可扩展性：Docker 使通过在不同的机器或云实例上运行多个容器来扩展你的应用程序变得容易。
- 可移植性：容器可以轻松地在环境之间移动，使在不同平台上部署你的应用程序变得简单。

要安装 Docker，请按照 [官方网站](https://www.docker.com/get-started) 上的说明进行操作。安装 Docker 后，你可以在 NestJS 项目中创建一个 `Dockerfile` 来定义构建你的容器镜像的步骤。

`Dockerfile` 是一个文本文件，包含 Docker 用于构建你的容器镜像的指令。

以下是一个 NestJS 应用程序的示例 Dockerfile：

```bash
# 使用官方 Node.js 镜像作为基础镜像
FROM node:20

# 设置容器内的工作目录
WORKDIR /usr/src/app

# 复制 package.json 和 package-lock.json 到工作目录
COPY package*.json ./

# 安装应用程序依赖项
RUN npm install

# 复制应用程序的其他文件
COPY . .

# 构建 NestJS 应用程序
RUN npm run build

# 暴露应用程序端口
EXPOSE 3000

# 运行应用程序的命令
CMD ["node", "dist/main"]
```

> 信息 **提示** 确保将 `node:20` 替换为你项目中使用的适当 Node.js 版本。你可以在 [官方 Docker Hub 仓库](https://hub.docker.com/_/node) 上找到可用的 Node.js Docker 镜像。

这是一个基本的 Dockerfile，它设置了 Node.js 环境，安装了应用程序依赖项，构建了 NestJS 应用程序，并运行了它。你可以根据项目需求自定义这个文件（例如，使用不同的基础镜像，优化构建过程，仅安装生产依赖项等）。

我们还需要创建一个 `.dockerignore` 文件，指定 Docker 在构建镜像时应忽略哪些文件和目录。在项目根目录中创建一个 `.dockerignore` 文件：

```bash
node_modules
dist
*.log
*.md
.git
```

这个文件确保不必要的文件不包含在容器镜像中，保持其轻量级。现在你已经设置了 Dockerfile，你可以构建你的 Docker 镜像。打开你的终端，导航到你的项目目录，并运行以下命令：

```bash
docker build -t my-nestjs-app .
```

在这个命令中：

- `-t my-nestjs-app`：用名称 `my-nestjs-app` 标记镜像。
- `.`：表示当前目录作为构建上下文。

构建镜像后，你可以将其作为容器运行。执行以下命令：

```bash
docker run -p 3000:3000 my-nestjs-app
```

在这个命令中：

- `-p 3000:3000`：将主机机器上的端口 3000 映射到容器中的端口 3000。
- `my-nestjs-app`：指定要运行的镜像。

你的 NestJS 应用程序现在应该在 Docker 容器中运行。

如果你想将你的 Docker 镜像部署到云提供商或与他人共享，你需要将其推送到 Docker 仓库（如 [Docker Hub](https://hub.docker.com/)、[AWS ECR](https://aws.amazon.com/ecr/) 或 [Google Container Registry](https://cloud.google.com/container-registry)）。

一旦你决定使用一个仓库，你可以通过以下步骤推送你的镜像：

```bash
docker login # 登录到你的 Docker 仓库
docker tag my-nestjs-app your-dockerhub-username/my-nestjs-app # 标记你的镜像
docker push your-dockerhub-username/my-nestjs-app # 推送你的镜像
```

将 `your-dockerhub-username` 替换为你的 Docker Hub 用户名或适当的仓库 URL。推送镜像后，你可以在任何机器上拉取它并将其作为容器运行。

像 AWS、Azure 和 Google Cloud 这样的云提供商提供了管理容器服务，简化了在规模上部署和管理容器的过程。这些服务提供了自动扩展、负载均衡和监控等功能，使你更容易在生产中运行你的 NestJS 应用程序。

#### 使用 Mau 轻松部署

[Mau](https://mau.nestjs.com/ 'Deploy Nest') 是我们在 [AWS](https://aws.amazon.com/) 上部署 NestJS 应用程序的官方平台。如果你还没有准备好手动管理你的基础设施（或者只是想节省时间），Mau 是完美的解决方案。

有了 Mau，配置和维护你的基础设施就像点击几个按钮一样简单。Mau 设计得简单直观，这样你就可以专注于构建应用程序，而不用担心底层基础设施。在幕后，我们使用 **Amazon Web Services** 为你提供一个功能强大且可靠的平台，同时抽象掉了 AWS 的所有复杂性。我们为你承担了所有的繁重工作，这样你就可以专注于构建应用程序和发展你的业务。

[Mau](https://mau.nestjs.com/ 'Deploy Nest') 对于初创企业、中小型企业、大型企业和想要快速启动而无需花费大量时间学习和管理基础设施的开发人员来说都是完美的。

它非常容易使用，你可以在几分钟内让你的基础设施运行起来。它还利用 AWS 作为后盾，让你在没有管理其复杂性的情况下获得 AWS 的所有优势。

<figure><img src="/assets/mau-metrics.png" /></figure>

有了 [Mau](https://mau.nestjs.com/ 'Deploy Nest')，你可以：

- 用几个点击部署你的 NestJS 应用程序（API、微服务等）。
- 配置数据库，如：
  - PostgreSQL
  - MySQL
  - MongoDB (DocumentDB)
  - Redis
  - 更多
- 设置代理服务，如：
  - RabbitMQ
  - Kafka
  - NATS
- 部署定时任务（**CRON 作业**）和后台工作进程。
- 部署 lambda 函数和无服务器应用程序。
- 设置 **CI/CD 管道** 以实现自动部署。
- 等等！

要使用 Mau 部署你的 NestJS 应用程序，只需运行以下命令：

```bash
$ npm install -g @nestjs/mau
$ mau deploy
```

今天就注册并[与 Mau 一起部署](https://mau.nestjs.com/ 'Deploy Nest')，让你的 NestJS 应用程序在几分钟内在 AWS 上运行！