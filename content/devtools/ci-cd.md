### CI/CD 集成

> 信息 **提示** 本章节涵盖了 Nest Devtools 与 Nest 框架的集成。如果您正在寻找 Devtools 应用程序，请访问 [Devtools](https://devtools.nestjs.com) 网站。

CI/CD 集成适用于拥有 **[企业版]** 计划的用户。

您可以通过观看此视频了解为什么及如何 CI/CD 集成可以帮助您：

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/r5RXcBrnEQ8"
    title="YouTube 视频播放器"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### 发布图表

首先，让我们配置应用程序启动文件（`main.ts`），以使用 `GraphPublisher` 类（从 `@nestjs/devtools-integration` 导出 - 详见前一章节以获取更多详细信息），如下所示：

```typescript
async function bootstrap() {
  const shouldPublishGraph = process.env.PUBLISH_GRAPH === "true";

  const app = await NestFactory.create(AppModule, {
    snapshot: true,
    preview: shouldPublishGraph,
  });

  if (shouldPublishGraph) {
    await app.init();

    const publishOptions = { ... } // 注意：这个选项对象将根据您的 CI/CD 提供商而变化
    const graphPublisher = new GraphPublisher(app);
    await graphPublisher.publish(publishOptions);

    await app.close();
  } else {
    await app.listen(process.env.PORT ?? 3000);
  }
}
```

如我们所见，我们在这里使用 `GraphPublisher` 来发布我们的序列化图表到集中式注册表。`PUBLISH_GRAPH` 是一个自定义环境变量，它将让我们控制是否应该发布图表（CI/CD 工作流），或者不发布（常规应用程序启动）。此外，我们在这里设置了 `preview` 属性为 `true`。有了这个标志，我们的应用程序将在预览模式下启动 - 这基本上意味着我们应用程序中的所有控制器、增强器和提供者的构造函数（和生命周期钩子）将不会被执行。注意 - 这不是 **必需的**，但对我们来说更简单，因为在这种情况下，我们在 CI/CD 管道中运行应用程序时，实际上不需要连接到数据库等。

`publishOptions` 对象将根据您的 CI/CD 提供商而变化。我们将在后续部分为您提供最流行 CI/CD 提供商的说明。

一旦图表成功发布，您将在工作流视图中看到以下输出：

<figure><img src="/assets/devtools/graph-published-terminal.png" /></figure>

每次我们的图表发布后，我们应该在项目的相应页面中看到一个新的条目：

<figure><img src="/assets/devtools/project.png" /></figure>

#### 报告

Devtools 为每个构建生成报告 **如果** 中央注册表中已经存储了相应的快照。例如，如果您为 `master` 分支创建了一个 PR，而图表已经发布 - 那么应用程序将能够检测到差异并生成报告。否则，报告将不会被生成。

要查看报告，请导航到项目的相应页面（见组织）。

<figure><img src="/assets/devtools/report.png" /></figure>

这在识别代码审查中可能未被注意到的更改时特别有用。例如，假设有人更改了一个 **深层嵌套提供者** 的范围。这个更改可能不会立即明显给审查者，但有了 Devtools，我们可以轻松发现这样的更改，并确保它们是故意的。或者，如果我们从特定端点移除了一个守卫，它将显示在报告中作为受影响的部分。现在，如果我们没有为该路由集成或端到端测试，我们可能不会注意到它不再受保护，而当我们这样做时，可能为时已晚。

同样，如果我们正在处理一个 **大型代码库** 并且我们修改了一个模块使其成为全局的，我们将看到图表中添加了多少条边，这在大多数情况下 - 是我们做错了事情的标志。

#### 构建预览

对于每个发布的图表，我们可以通过点击 **预览** 按钮来回溯查看它以前的样子。此外，如果生成了报告，我们应该会在图表上看到差异被突出显示：

- 绿色节点代表添加的元素
- 浅白色节点代表更新的元素
- 红色节点代表删除的元素

见下图：

<figure><img src="/assets/devtools/nodes-selection.png" /></figure>

回溯的能力让您可以通过比较当前图表与之前的图表来调查和解决问题。根据您如何设置，每个拉取请求（甚至每个提交）都将在注册表中有一个相应的快照，所以您可以轻松地回溯并查看发生了什么变化。将 Devtools 视为一个理解 Nest 如何构建您的应用程序图表的 Git，并且具有 **可视化** 它的能力。

#### 集成：GitHub Actions

首先，让我们从在项目的 `.github/workflows` 目录中创建一个新的 GitHub 工作流开始，并称之为，例如，`publish-graph.yml`。在这个文件中，让我们使用以下定义：

```yaml
name: Devtools

on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - '*'

jobs:
  publish:
    if: github.actor!= 'dependabot[bot]'
    name: 发布图表
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'
      - name: 安装依赖
        run: npm ci
      - name: 设置环境（PR）
        if: ${{ github.event_name == 'pull_request' }}
        shell: bash
        run: |
          echo "COMMIT_SHA=${{ github.event.pull_request.head.sha }}" >>${GITHUB_ENV}
      - name: 设置环境（Push）
        if: ${{ github.event_name == 'push' }}
        shell: bash
        run: |
          echo "COMMIT_SHA=${GITHUB_SHA}" >> ${GITHUB_ENV}
      - name: 发布
        run: PUBLISH_GRAPH=true npm run start
        env:
          DEVTOOLS_API_KEY: CHANGE_THIS_TO_YOUR_API_KEY
          REPOSITORY_NAME: ${{ github.event.repository.name }}
          BRANCH_NAME: ${{ github.head_ref || github.ref_name }}
          TARGET_SHA: ${{ github.event.pull_request.base.sha }}
```

理想情况下，`DEVTOOLS_API_KEY` 环境变量应该从 GitHub Secrets 中检索，更多信息请[点击这里](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository)。

此工作流将在针对 `master` 分支的每个拉取请求 OR 直接提交到 `master` 分支时运行。请随意根据您的项目需求调整此配置。这里最重要的是我们为我们的 `GraphPublisher` 类提供了必要的环境变量（以运行）。

然而，在我们开始使用此工作流之前，有一个变量需要更新 - `DEVTOOLS_API_KEY`。我们可以在此[页面](https://devtools.nestjs.com/settings/manage-api-keys)为我们的项目生成一个专用的 API 密钥。

最后，让我们再次导航到 `main.ts` 文件并更新我们之前留下的空 `publishOptions` 对象。

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

为了获得最佳的开发者体验，请确保通过点击“集成 GitHub 应用”按钮（见下图）为您的项目集成 GitHub 应用程序。注意 - 这不是必需的。

<figure><img src="/assets/devtools/integrate-github-app.png" /></figure>

有了这个集成，您将能够在您的拉取请求中直接看到预览/报告生成过程的状态：

<figure><img src="/assets/devtools/actions-preview.png" /></figure>

#### 集成：Gitlab Pipelines

首先，让我们从在项目的根目录中创建一个新的 Gitlab CI 配置文件开始，并称之为，例如，`.gitlab-ci.yml`。在这个文件中，让我们使用以下定义：

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

> 信息 **提示** 理想情况下，`DEVTOOLS_API_KEY` 环境变量应该从 secrets 中检索。

此工作流将在针对 `master` 分支的每个拉取请求 OR 直接提交到 `master` 分支时运行。请随意根据您的项目需求调整此配置。这里最重要的是我们为我们的 `GraphPublisher` 类提供了必要的环境变量（以运行）。

然而，在我们开始使用此工作流之前，有一个变量（在此工作流定义中）需要更新 - `DEVTOOLS_API_KEY`。我们可以在此**页面**为我们的项目生成一个专用的 API 密钥。

最后，让我们再次导航到 `main.ts` 文件并更新我们之前留下的空 `publishOptions` 对象。

```yaml
image: node:16

stages:
  - build

cache:
  key:
    files:
      - package-lock.json
  paths:
    - node_modules/

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: always
    - if: $CI_COMMIT_BRANCH == "master" && $CI_PIPELINE_SOURCE == "push"
      when: always
    - when: never

install_dependencies:
  stage: build
  script:
    - npm ci

publish_graph:
  stage: build
  needs:
    - install_dependencies
  script: npm run start
  variables:
    PUBLISH_GRAPH: 'true'
    DEVTOOLS_API_KEY: 'CHANGE_THIS_TO_YOUR_API_KEY'
```

#### 其他 CI/CD 工具

Nest Devtools CI/CD 集成可以与您选择的任何 CI/CD 工具一起使用（例如，[Bitbucket Pipelines](https://bitbucket.org/product/features/pipelines)，[CircleCI](https://circleci.com/) 等），因此不要觉得自己局限于我们在这里描述的提供商。

查看以下 `publishOptions` 对象配置，以了解发布给定提交/构建/PR 图表所需的信息。

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.CI_PROJECT_NAME,
  owner: process.env.CI_PROJECT_ROOT_NAMESPACE,
  sha: process.env.CI_COMMIT_SHA,
  target: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA,
  trigger: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA ? 'pull' : 'push',
  branch: process.env.CI_COMMIT_BRANCH ?? process.env.CI_MERGE_REQUEST_SOURCE_BRANCH_NAME,
};
```

这些信息中的大部分是通过 CI/CD 内置环境变量提供的（参见 [CircleCI 内置环境变量列表](https://circleci.com/docs/variables/#built-in-environment-variables) 和 [Bitbucket 变量](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/)）。

在发布图表的管道配置方面，我们建议使用以下触发器：

- `push` 事件 - 仅当当前分支代表部署环境时，例如 `master`，`main`，`staging`，`production` 等。
- `pull request` 事件 - 总是，或者当 **目标分支** 代表部署环境时（见上文）。