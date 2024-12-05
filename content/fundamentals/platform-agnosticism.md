### 平台无关性

Nest 是一个平台无关的框架。这意味着你可以开发**可重用的逻辑部分**，这些部分可以在不同类型的应用程序中使用。例如，大多数组件可以在不同的底层 HTTP 服务器框架（例如，Express 和 Fastify）之间无需更改地重用，甚至可以在不同类型的应用程序（例如，HTTP 服务器框架、具有不同传输层的微服务和 WebSocket）之间重用。

#### 一次构建，随处使用

文档的**概览**部分主要展示了使用 HTTP 服务器框架（例如，提供 REST API 的应用程序或提供 MVC 风格的服务器端渲染应用程序）的编码技术。然而，所有这些构建块都可以在不同的传输层上使用（[微服务](/microservices/basics)或[websockets](/websockets/gateways)）。

此外，Nest 还提供了一个专门的[GraphQL](/graphql/quick-start)模块。你可以将 GraphQL 作为你的 API 层，与提供 REST API 互换使用。

而且，[应用上下文](/application-context)功能有助于在 Nest 上创建任何类型的 Node.js 应用程序——包括 CRON 作业和 CLI 应用程序。

Nest 致力于成为一个全面的 Node.js 应用程序平台，为你的应用程序带来更高级别的模块化和可重用性。一次构建，随处使用！