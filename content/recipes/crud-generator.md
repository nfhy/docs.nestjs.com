### CRUD 生成器（仅限 TypeScript）

在项目生命周期中，当我们构建新功能时，通常需要向我们的应用程序添加新资源。这些资源通常需要多次重复的操作，每次定义新资源时我们都必须重复这些操作。

#### 引言

让我们想象一个现实世界的场景，我们需要为两个实体，比如说 **用户** 和 **产品** 实体，暴露 CRUD 端点。

遵循最佳实践，对于每个实体，我们需要执行以下几项操作：

- 生成模块（`nest g mo`），以保持代码组织并建立清晰的边界（将相关组件分组）
- 生成控制器（`nest g co`），以定义 CRUD 路由（或 GraphQL 应用的查询/变更）
- 生成服务（`nest g s`），以实现和隔离业务逻辑
- 生成实体类/接口以表示资源数据形状
- 生成数据传输对象（或 GraphQL 应用的输入）以定义数据如何在网络中传输

这真是很多步骤！

为了帮助加快这一重复过程，[Nest CLI](/cli/overview) 提供了一个生成器（schematic），它自动生成所有样板代码，帮助我们避免做所有这些工作，并使开发人员体验更简单。

> 信息 **注意** 生成器支持生成 **HTTP** 控制器，**微服务** 控制器，**GraphQL** 解析器（代码优先和模式优先）以及 **WebSocket** 网关。

#### 生成新资源

要创建新资源，只需在项目的根目录中运行以下命令：

```shell
$ nest g resource
```

`nest g resource` 命令不仅生成所有 NestJS 构建块（模块、服务、控制器类），还生成实体类、DTO 类以及测试（`.spec`）文件。

下面你可以看到生成的控制器文件（用于 REST API）：

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.usersService.update(+id, updateUserDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(+id);
  }
}
```

同时，它自动为所有 CRUD 端点创建占位符（REST API 的路由，GraphQL 的查询和变更，微服务和 WebSocket 网关的消息订阅） - 所有这些都不需要你动手。

> 警告 **注意** 生成的服务类 **不** 绑定到任何特定的 **ORM（或数据源）**。这使得生成器足够通用，以满足任何项目的需求。默认情况下，所有方法都将包含占位符，允许你用特定于你项目的数据源填充它。

同样，如果你想为 GraphQL 应用生成解析器，只需选择 `GraphQL (code first)`（或 `GraphQL (schema first)`）作为你的传输层。

在这种情况下，NestJS 将生成解析器类而不是 REST API 控制器：

```shell
$ nest g resource users

> ? 您使用什么传输层？GraphQL (code first)
> ? 您想生成 CRUD 入口点吗？是
> CREATE src/users/users.module.ts (224 bytes)
> CREATE src/users/users.resolver.spec.ts (525 bytes)
> CREATE src/users/users.resolver.ts (1109 bytes)
> CREATE src/users/users.service.spec.ts (453 bytes)
> CREATE src/users/users.service.ts (625 bytes)
> CREATE src/users/dto/create-user.input.ts (195 bytes)
> CREATE src/users/dto/update-user.input.ts (281 bytes)
> CREATE src/users/entities/user.entity.ts (187 bytes)
> UPDATE src/app.module.ts (312 bytes)
```

> 信息 **提示** 为了避免生成测试文件，你可以传递 `--no-spec` 标志，如下所示：`nest g resource users --no-spec`

我们可以看到，不仅创建了所有样板变更和查询，而且一切都紧密联系在一起。我们正在使用 `UsersService`、`User` 实体和我们的 DTO。