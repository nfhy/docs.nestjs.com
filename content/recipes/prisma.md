### Prisma

[Prisma](https://www.prisma.io) 是一个用于 Node.js 和 TypeScript 的[开源](https://github.com/prisma/prisma) ORM（对象关系映射）。它被用作编写纯 SQL 或使用其他数据库访问工具（如 SQL 查询构建器（例如 [knex.js](https://knexjs.org/)）或 ORM（例如 [TypeORM](https://typeorm.io/) 和 [Sequelize](https://sequelize.org/)）的**替代品**。Prisma 目前支持 PostgreSQL、MySQL、SQL Server、SQLite 和 CockroachDB（[预览版](https://www.prisma.io/docs/reference/database-reference/supported-databases)）。

虽然 Prisma 可以与纯 JavaScript 一起使用，但它支持 TypeScript 并提供了超越 TypeScript 生态系统中其他 ORM 的类型安全性。你可以在这里找到 Prisma 和 TypeORM 类型安全性保证的深入比较：[Prisma 和 TypeORM](https://www.prisma.io/docs/concepts/more/comparisons/prisma-and-typeorm#type-safety)。

> **注意**：如果你想快速了解 Prisma 的工作原理，可以按照[快速入门](https://www.prisma.io/docs/getting-started/quickstart)或阅读[文档](https://www.prisma.io/docs/)中的[介绍](https://www.prisma.io/docs/understand-prisma/introduction)。在[`prisma-examples`](https://github.com/prisma/prisma-examples/)仓库中，还有为[REST](https://github.com/prisma/prisma-examples/tree/latest/orm/rest-nestjs)和[GraphQL](https://github.com/prisma/prisma-examples/tree/latest/orm/graphql-nestjs)准备的即用型示例。

#### 开始使用

在这个指南中，你将学习如何从零开始使用 NestJS 和 Prisma。你将构建一个带有 REST API 的示例 NestJS 应用程序，该应用程序可以在数据库中读写数据。

为了本指南的目的，你将使用一个[SQLite](https://sqlite.org/)数据库，以减少设置数据库服务器的开销。请注意，即使你使用的是 PostgreSQL 或 MySQL，你仍然可以按照本指南进行操作，在适当的地方会有使用这些数据库的额外说明。

> **注意**：如果你已经有一个现有项目，并考虑迁移到 Prisma，可以按照[向现有项目添加 Prisma](https://www.prisma.io/docs/getting-started/setup-prisma/add-to-existing-project-typescript-postgres)的指南进行操作。如果你从 TypeORM 迁移过来，可以阅读[从 TypeORM 迁移到 Prisma](https://www.prisma.io/docs/guides/migrate-to-prisma/migrate-from-typeorm)的指南。

#### 创建你的 NestJS 项目

要开始，请安装 NestJS CLI 并使用以下命令创建你的应用程序骨架：

```bash
$ npm install -g @nestjs/cli
$ nest new hello-prisma
```

查看[第一步](https://docs.nestjs.com/first-steps)页面，了解更多关于此命令创建的项目文件。注意，你现在可以运行 `npm start` 来启动你的应用程序。目前在 `http://localhost:3000/` 上运行的 REST API 提供了一个在 `src/app.controller.ts` 中实现的单一路由。在这个指南的过程中，你将实现额外的路由来存储和检索关于 _用户_ 和 _帖子_ 的数据。

#### 设置 Prisma

首先，将 Prisma CLI 作为开发依赖项安装到你的项目中：

```bash
$ cd hello-prisma
$ npm install prisma --save-dev
```

在接下来的步骤中，我们将使用[Prisma CLI](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-cli)。作为最佳实践，建议通过在前面加上 `npx` 来本地调用 CLI：

```bash
$ npx prisma
```

<details><summary>如果你使用 Yarn，点击展开</summary>

如果你使用 Yarn，则可以按照以下方式安装 Prisma CLI：

```bash
$ yarn add prisma --dev
```

安装后，你可以通过在前面加上 `yarn` 来调用它：

```bash
$ yarn prisma
```

</details>

现在，使用 Prisma CLI 的 `init` 命令创建你的初始 Prisma 设置：

```bash
$ npx prisma init
```

此命令创建了一个新的 `prisma` 目录，包含以下内容：

- `schema.prisma`：指定你的数据库连接，并包含数据库模式
- `.env`：一个[dotenv](https://github.com/motdotla/dotenv)文件，通常用于将你的数据库凭据存储在一组环境变量中

#### 设置数据库连接

你的数据库连接在 `schema.prisma` 文件的 `datasource` 块中配置。默认设置为 `postgresql`，但由于本指南中你使用的是 SQLite 数据库，你需要调整 `datasource` 块的 `provider` 字段为 `sqlite`：

```groovy
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

现在，打开 `.env` 并调整 `DATABASE_URL` 环境变量如下所示：

```bash
DATABASE_URL="file:./dev.db"
```

确保你配置了[ConfigModule](https://docs.nestjs.com/techniques/configuration)，否则 `DATABASE_URL` 变量将无法从 `.env` 中获取。

SQLite 数据库是简单的文件；使用 SQLite 数据库不需要服务器。因此，你不需要配置一个带有 _主机_ 和 _端口_ 的连接 URL，而是可以直接指向一个本地文件，在这个例子中文件名为 `dev.db`。此文件将在下一步中创建。

<details><summary>如果你使用 PostgreSQL、MySQL、MsSQL 或 Azure SQL，点击展开</summary>

使用 PostgreSQL 和 MySQL 时，你需要配置连接 URL 指向 _数据库服务器_。你可以在这里了解更多关于所需连接 URL 格式的信息：[连接 URL](https://www.prisma.io/docs/reference/database-reference/connection-urls)。

**PostgreSQL**

如果你使用 PostgreSQL，你需要按照以下方式调整 `schema.prisma` 和 `.env` 文件：

**`schema.prisma`**

```groovy
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

**`.env`**

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=SCHEMA"
```

用你的数据库凭据替换所有大写字母的占位符。如果你不确定 `SCHEMA` 占位符应该提供什么，它很可能是默认值 `public`：

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=public"
```

如果你想了解如何设置 PostgreSQL 数据库，可以按照这篇关于[在 Heroku 上设置免费的 PostgreSQL 数据库](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1)的指南。

**MySQL**

如果你使用 MySQL，你需要按照以下方式调整 `schema.prisma` 和 `.env` 文件：

**`schema.prisma`**

```groovy
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

**`.env`**

```bash
DATABASE_URL="mysql://USER:PASSWORD@HOST:PORT/DATABASE"
```

用你的数据库凭据替换所有大写字母的占位符。

**Microsoft SQL Server / Azure SQL Server**

如果你使用 Microsoft SQL Server 或 Azure SQL Server，你需要按照以下方式调整 `schema.prisma` 和 `.env` 文件：

**`schema.prisma`**

```groovy
datasource db {
  provider = "sqlserver"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

**`.env`**

用你的数据库凭据替换所有大写字母的占位符。如果你不确定 `encrypt` 占位符应该提供什么，它很可能是默认值 `true`：

```bash
DATABASE_URL="sqlserver://HOST:PORT;database=DATABASE;user=USER;password=PASSWORD;encrypt=true"
```

</details>

#### 使用 Prisma Migrate 创建两个数据库表

在本节中，你将使用[Prisma Migrate](https://www.prisma.io/docs/concepts/components/prisma-migrate)在数据库中创建两个新表。Prisma Migrate 为你在 Prisma 模式中的声明式数据模型定义生成 SQL 迁移文件。这些迁移文件是完全可定制的，以便你可以配置底层数据库的任何附加功能或包含额外命令，例如用于种子数据。

将以下两个模型添加到你的 `schema.prisma` 文件中：

```groovy
model User {
  id    Int     @default(autoincrement()) @id
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int      @default(autoincrement()) @id
  title     String
  content   String?
  published Boolean? @default(false)
  author    User?    @relation(fields: [authorId], references: [id])
  authorId  Int?
}
```

有了你的 Prisma 模型，你可以生成 SQL 迁移文件并对数据库运行它们。在终端中运行以下命令：

```bash
$ npx prisma migrate dev --name init
```

这个 `prisma migrate dev` 命令生成 SQL 文件，并直接对数据库运行它们。在这个例子中，以下迁移文件在现有的 `prisma` 目录中创建：

```bash
$ tree prisma
prisma
├── dev.db
├── migrations
│   └── 20201207100915_init
│       └── migration.sql
└── schema.prisma
```

<details><summary>展开查看生成的 SQL 语句</summary>

在你的 SQLite 数据库中创建了以下表：

```sql
-- CreateTable
CREATE TABLE "User" (
    "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    "email" TEXT NOT NULL,
    "name" TEXT
);

-- CreateTable
CREATE TABLE "Post" (
    "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    "title" TEXT NOT NULL,
    "content" TEXT,
    "published" BOOLEAN DEFAULT false,
    "authorId" INTEGER,

    FOREIGN KEY ("authorId") REFERENCES "User"("id") ON DELETE SET NULL ON UPDATE CASCADE
);

-- CreateIndex
CREATE UNIQUE INDEX "User.email_unique" ON "User"("email");
```

</details>

#### 安装并生成 Prisma 客户端

Prisma 客户端是一个类型安全的数据库客户端，它是根据你的 Prisma 模型定义*生成*的。通过这种方法，Prisma 客户端可以暴露[CRUD](https://www.prisma.io/docs/concepts/components/prisma-client/crud)操作，这些操作专门针对你的模型进行*定制*。

要在项目中安装 Prisma 客户端，请在终端中运行以下命令：

```bash
$ npm install @prisma/client
```

注意，在安装过程中，Prisma 会自动为你调用 `prisma generate` 命令。将来，每次更改 Prisma 模型后，你都需要运行此命令以更新生成的 Prisma 客户端。

> **注意**：`prisma generate` 命令读取你的 Prisma 模式，并更新 `node_modules/@prisma/client` 中的生成 Prisma 客户端库。

#### 在你的 NestJS 服务中使用 Prisma 客户端

你现在可以使用 Prisma 客户端发送数据库查询。如果你想更多地了解如何使用 Prisma 客户端构建查询，请查看[API 文档](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-client/crud)。

在设置你的 NestJS 应用程序时，你将希望在服务中抽象出 Prisma 客户端 API 的数据库查询。首先，你可以创建一个新的 `PrismaService`，负责实例化 `PrismaClient` 并连接到你的数据库。

在 `src` 目录中，创建一个名为 `prisma.service.ts` 的新文件，并添加以下代码：

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }
}
```

> **注意**：`onModuleInit` 是可选的——如果你省略它，Prisma 将在第一次调用数据库时延迟连接。

接下来，你可以编写服务，用于为 Prisma 模式中的 `User` 和 `Post` 模型进行数据库调用。

仍在 `src` 目录中，创建一个名为 `user.service.ts` 的新文件，并添加以下代码：

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { User, Prisma } from '@prisma/client';

@Injectable()
export class UserService {
  constructor(private prisma: PrismaService) {}

  async user(
    userWhereUniqueInput: Prisma.UserWhereUniqueInput,
  ): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: userWhereUniqueInput,
    });
  }

  async users(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.UserWhereUniqueInput;
    where?: Prisma.UserWhereInput;
    orderBy?: Prisma.UserOrderByWithRelationInput;
  }): Promise<User[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.user.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createUser(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({
      data,
    });
  }

  async updateUser(params: {
    where: Prisma.UserWhereUniqueInput;
    data: Prisma.UserUpdateInput;
  }): Promise<User> {
    const { where, data } = params;
    return this.prisma.user.update({
      data,
      where,
    });
  }

  async deleteUser(where: Prisma.UserWhereUniqueInput): Promise<User> {
    return this.prisma.user.delete({
      where,
    });
  }
}
```

注意你是如何使用 Prisma 客户端的生成类型来确保服务公开的方法是正确类型的。因此，你节省了为你的模型输入类型编写额外接口或 DTO 文件的样板代码。

现在对 `Post` 模型做同样的操作。

仍在 `src` 目录中，创建一个名为 `post.service.ts` 的新文件，并添加以下代码：

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { Post, Prisma } from '@prisma/client';

@Injectable()
export class PostService {
  constructor(private prisma: PrismaService) {}

  async post(
    postWhereUniqueInput: Prisma.PostWhereUniqueInput,
  ): Promise<Post | null> {
    return this.prisma.post.findUnique({
      where: postWhereUniqueInput,
    });
  }

  async posts(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.PostWhereUniqueInput;
    where?: Prisma.PostWhereInput;
    orderBy?: Prisma.PostOrderByWithRelationInput;
  }): Promise<Post[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.post.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createPost(data: Prisma.PostCreateInput): Promise<Post> {
    return this.prisma.post.create({
      data,
    });
  }

  async updatePost(params: {
    where: Prisma.PostWhereUniqueInput;
    data: Prisma.PostUpdateInput;
  }): Promise<Post> {
    const { data, where } = params;
    return this.prisma.post.update({
      data,
      where,
    });
  }

  async deletePost(where: Prisma.PostWhereUniqueInput): Promise<Post> {
    return this.prisma.post.delete({
      where,
    });
  }
}
```

你的 `UserService` 和 `PostService` 目前包装了 Prisma 客户端中可用的 CRUD 查询。在现实世界的应用中，服务也是添加应用程序业务逻辑的地方。例如，你可以在 `UserService` 中有一个叫做 `updatePassword` 的方法，负责更新用户的密码。

记得在应用模块中注册新服务。

##### 在主应用控制器中实现你的 REST API 路由

最后，你将使用上一节中创建的服务来实现应用程序的不同路由。为了本指南的目的，你将把所有路由放入已存在的 `AppController` 类中。

用以下代码替换 `app.controller.ts` 文件的内容：

```typescript
import {
  Controller,
  Get,
  Param,
  Post,
  Body,
  Put,
  Delete,
} from '@nestjs/common';
import { UserService } from './user.service';
import { PostService } from './post.service';
import { User as UserModel, Post as PostModel } from '@prisma/client';

@Controller()
export class AppController {
  constructor(
    private readonly userService: UserService,
    private readonly postService: PostService,
  ) {}

  @Get('post/:id')
  async getPostById(@Param('id') id: string): Promise<PostModel> {
    return this.postService.post({ id: Number(id) });
  }

  @Get('feed')
  async getPublishedPosts(): Promise<PostModel[]> {
    return this.postService.posts({
      where: { published: true },
    });
  }

  @Get('filtered-posts/:searchString')
  async getFilteredPosts(
    @Param('searchString') searchString: string,
  ): Promise<PostModel[]> {
    return this.postService.posts({
      where: {
        OR: [
          {
            title: { contains: searchString },
          },
          {
            content: { contains: searchString },
          },
        ],
      },
    });
  }

  @Post('post')
  async createDraft(
    @Body() postData: { title: string; content?: string; authorEmail: string },
  ): Promise<PostModel> {
    const { title, content, authorEmail } = postData;
    return this.postService.createPost({
      title,
      content,
      author: {
        connect: { email: authorEmail },
      },
    });
  }

  @Post('user')
  async signupUser(
    @Body() userData: { name?: string; email: string },
  ): Promise<UserModel> {
    return this.userService.createUser(userData);
  }

  @Put('publish/:id')
  async publishPost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.updatePost({
      where: { id: Number(id) },
      data: { published: true },
    });
  }

  @Delete('post/:id')
  async deletePost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.deletePost({ id: Number(id) });
  }
}
```

这个控制器实现了以下路由：

###### `GET`

- `/post/:id`：根据其 `id` 获取单个帖子
- `/feed`：获取所有已发布的帖子
- `/filter-posts/:searchString`：根据 `title` 或 `content` 过滤帖子

###### `POST`

- `/post`：创建一个新的帖子
  - 正文：
    - `title: String`（必需）：帖子的标题
    - `content: String`（可选）：帖子的内容
    - `authorEmail: String`（必需）：创建帖子的用户的电子邮件
- `/user`：创建一个新用户
  - 正文：
    - `email: String`（必需）：用户的电子邮件地址
    - `name: String`（可选）：用户的名字

###### `PUT`

- `/publish/:id`：通过其 `id` 发布一个帖子

###### `DELETE`

- `/post/:id`：根据其 `id` 删除一个帖子

#### 总结

在这个指南中，你学习了如何使用 Prisma 和 NestJS 实现 REST API。实现 API 路由的控制器调用 `PrismaService`，后者又使用 Prisma 客户端向数据库发送查询以满足传入请求的数据需求。

如果你想更多地了解如何将 NestJS 与 Prisma 结合使用，请务必查看以下资源：

- [NestJS & Prisma](https://www.prisma.io/nestjs)
- [REST & GraphQL 的即用型示例项目](https://github.com/prisma/prisma-examples/)
- [生产就绪的启动套件](https://github.com/notiz-dev/nestjs-prisma-starter#instructions)
- [视频：使用 Prisma 和 NestJS 访问数据库（5分钟）](https://www.youtube.com/watch?v=UlVJ340UEuk&ab_channel=Prisma)，由 [Marc Stammerjohann](https://github.com/marcjulian) 提供