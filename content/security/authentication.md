### 认证

认证是大多数应用程序的**基本**部分。处理认证有许多不同的方法和策略。对于任何项目采取的方法取决于其特定的应用需求。本章介绍了几种可以适应不同需求的认证方法。

让我们详细阐述我们的需求。在这个用例中，客户端将首先通过用户名和密码进行认证。一旦认证成功，服务器将颁发一个JWT，该JWT可以在后续请求的授权头部作为[承载令牌](https://tools.ietf.org/html/rfc6750)发送，以证明认证。我们还将创建一个受保护的路由，只有包含有效JWT的请求才能访问。

我们将从第一个需求开始：认证用户。然后我们将通过颁发JWT来扩展这一点。最后，我们将创建一个受保护的路由，在请求上检查有效的JWT。

#### 创建认证模块

我们首先生成一个`AuthModule`，其中包含一个`AuthService`和一个`AuthController`。我们将使用`AuthService`来实现认证逻辑，使用`AuthController`来暴露认证端点。

```bash
$ nest g module auth
$ nest g controller auth
$ nest g service auth
```

在我们实现`AuthService`时，我们发现将用户操作封装在`UsersService`中很有用，因此让我们现在生成该模块和服务：

```bash
$ nest g module users
$ nest g service users
```

将这些生成的文件的默认内容替换为以下内容。对于我们的示例应用程序，`UsersService`简单地维护一个硬编码的内存用户列表，以及一个按用户名检索的方法。在真实的应用程序中，这就是您构建用户模型和持久层的地方，使用您选择的库（例如，TypeORM、Sequelize、Mongoose等）。

```typescript
@@filename(users/users.service)
import { Injectable } from '@nestjs/common';

// 这应该是一个真实的类/接口，代表用户实体
export type User = any;

@Injectable()
export class UsersService {
  private readonly users = [
    {
      userId: 1,
      username: 'john',
      password: 'changeme',
    },
    {
      userId: 2,
      username: 'maria',
      password: 'guess',
    },
  ];

  async findOne(username: string): Promise<User | undefined> {
    return this.users.find(user => user.username === username);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  constructor() {
    this.users = [
      {
        userId: 1,
        username: 'john',
        password: 'changeme',
      },
      {
        userId: 2,
        username: 'maria',
        password: 'guess',
      },
    ];
  }

  async findOne(username) {
    return this.users.find(user => user.username === username);
  }
}
```

在`UsersModule`中，唯一需要更改的是将`UsersService`添加到`@Module`装饰器的exports数组中，以便它在模块外部可见（我们很快将在`AuthService`中使用它）。

```typescript
@@filename(users/users.module)
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
@@switch
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

#### 实现“登录”端点

我们的`AuthService`的工作是检索用户并验证密码。我们为此目的创建了一个`signIn()`方法。在下面的代码中，我们使用方便的ES6展开运算符从用户对象中剥离密码属性，然后再返回它。这是一种常见的做法，当返回用户对象时，因为您不想暴露密码或其他安全密钥等敏感字段。

```typescript
@@filename(auth/auth.service)
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  async signIn(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: 在这里生成一个JWT并返回它
    // 而不是用户对象
    return result;
  }
}
@@switch
import { Injectable, Dependencies, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
@Dependencies(UsersService)
export class AuthService {
  constructor(usersService) {
    this.usersService = usersService;
  }

  async signIn(username: string, pass: string) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: 在这里生成一个JWT并返回它
    // 而不是用户对象
    return result;
  }
}
```

> 警告 **警告** 当然，在真实的应用程序中，您不会以明文形式存储密码。您将使用像[bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme)这样的库，使用加盐的单向哈希算法。用这种方法，您只存储哈希密码，然后比较存储的密码与**传入**密码的哈希版本，从而从不以明文形式存储或暴露用户密码。为了保持我们的示例应用程序简单，我们违反了这一绝对命令，使用明文。**不要在您的实际应用程序中这样做！**

现在，我们更新我们的`AuthModule`以导入`UsersModule`。

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
```

有了这些，让我们打开`AuthController`并添加一个`signIn()`方法。这个方法将被客户端调用来认证用户。它将在请求正文中接收用户名和密码，并在用户认证后返回一个JWT令牌。

```typescript
@@filename(auth/auth.controller)
import { Body, Controller, Post, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }
}
```

> 信息 **提示** 理想情况下，我们应该使用DTO类来定义请求正文的形状，而不是使用`Record<string, any>`类型。有关更多信息，请参见[验证](/techniques/validation)章节。

<app-banner-courses-auth></app-banner-courses-auth>

#### JWT令牌

我们准备继续我们认证系统的JWT部分。让我们审查并完善我们的需求：

- 允许用户使用用户名/密码进行认证，并在后续调用受保护API端点时返回JWT。我们已经很好地满足了这个需求。为了完成它，我们需要编写颁发JWT的代码。
- 创建基于JWT作为承载令牌存在与否的API路由进行保护

我们需要安装一个额外的包来支持我们的JWT需求：

```bash
$ npm install --save @nestjs/jwt
```

> 信息 **提示** `@nestjs/jwt`包（了解更多[这里](https://github.com/nestjs/jwt))是一个实用程序包，它有助于JWT操作。这包括生成和验证JWT令牌。

为了保持我们的服务清晰模块化，我们将在`authService`中处理生成JWT。打开`auth`文件夹中的`auth.service.ts`文件，注入`JwtService`，并更新`signIn`方法以生成JWT令牌，如下所示：

```typescript
@@filename(auth/auth.service)
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async signIn(
    username: string,
    pass: string,
  ): Promise<{ access_token: string }> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { sub: user.userId, username: user.username };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
@@switch
import { Injectable, Dependencies, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Dependencies(UsersService, JwtService)
@Injectable()
export class AuthService {
  constructor(usersService, jwtService) {
    this.usersService = usersService;
    this.jwtService = jwtService;
  }

  async signIn(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
```

我们使用`@nestjs/jwt`库，它提供了一个`signAsync()`函数，从`user`对象属性的一个子集中生成我们的JWT，然后我们将其作为一个简单的对象返回，其中只有一个`access_token`属性。注意：我们选择`sub`属性名来保存我们的`userId`值，以与JWT标准保持一致。

现在我们需要更新`AuthModule`以导入新依赖并配置`JwtModule`。

首先，在`auth`文件夹中创建`constants.ts`，并添加以下代码：

```typescript
@@filename(auth/constants)
export const jwtConstants = {
  secret: 'DO NOT USE THIS VALUE. INSTEAD, CREATE A COMPLEX SECRET AND KEEP IT SAFE OUTSIDE OF THE SOURCE CODE.',
};
@@switch
export const jwtConstants = {
  secret: 'DO NOT USE THIS VALUE. INSTEAD, CREATE A COMPLEX SECRET AND KEEP IT SAFE OUTSIDE OF THE SOURCE CODE.',
};
```

我们将使用这个来共享我们的密钥在JWT签名和验证步骤之间。

> 警告 **警告** **不要公开暴露这个密钥**。我们在这里这样做是为了清楚地说明代码在做什么，但在生产系统中**您必须使用适当的措施保护这个密钥**，例如使用秘密库、环境变量或配置服务。

现在，打开`auth.module.ts`在`auth`文件夹，并更新它如下所示：

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

> 信息 **提示** 我们将`JwtModule`注册为全局的，以使我们的工作更轻松。这意味着我们不需要在应用程序的其他任何地方导入`JwtModule`。

我们使用`register()`配置`JwtModule`，传入一个配置对象。有关Nest `JwtModule`的更多信息，请参见[这里](https://github.com/nestjs/jwt/blob/master/README.md)，有关可用配置选项的更多详细信息，请参见[这里](https://github.com/auth0/node-jsonwebtoken#usage)。

让我们继续使用cURL测试我们的路由。您可以使用`UsersService`中硬编码的任何`user`对象进行测试。

```bash
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # 注意：上面的JWT已截断
```

#### 实现认证守卫

我们现在可以解决我们的最终需求：通过要求请求上存在有效的JWT来保护端点。我们将通过创建一个`AuthGuard`来实现这一点，我们可以将其用于保护我们的路由。

```typescript
@@filename(auth/auth.guard)
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { jwtConstants } from './constants';
import { Request } from 'express';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(
        token,
        {
          secret: jwtConstants.secret
        }
      );
      // 💡 我们在这里将payload分配给请求对象
      // 以便我们可以在路由处理程序中访问它
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

我们现在可以实施我们的受保护路由，并注册我们的`AuthGuard`来保护它。

打开`auth.controller.ts`文件，并如下所示进行更新：

```typescript
@@filename(auth.controller)
import {
  Body,
  Controller,
  Get,
  HttpCode,
  HttpStatus,
  Post,
  Request,
  UseGuards
} from '@nestjs/common';
import { AuthGuard } from './auth.guard';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }

  @UseGuards(AuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

我们应用了我们刚刚创建的`AuthGuard`到`GET /profile`路由，以便它将受到保护。

确保应用程序正在运行，并使用`cURL`测试路由。

```bash
$ # GET /profile
$ curl http://localhost:3000/auth/profile
{"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."}

$ # 使用前一步返回的access_token作为承载代码GET /profile
$ curl http://localhost:3000/auth/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
{"sub":1,"username":"john","iat":...,"exp":...}
```

请注意，在`AuthModule`中，我们将JWT配置为`60秒`的过期时间。这个过期时间太短了，处理令牌过期和刷新的细节超出了本文的范围。然而，我们选择这个是为了演示JWT的一个重要特性。如果您在认证后等待60秒再尝试`GET /auth/profile`请求，您将收到`401 Unauthorized`响应。这是因为`@nestjs/jwt`自动检查JWT的过期时间，节省了您在应用程序中这样做的麻烦。

我们已经完成了我们的JWT认证实现。JavaScript客户端（如Angular/React/Vue）和其他JavaScript应用程序现在可以认证并与我们的API服务器安全通信。

#### 全局启用认证

如果您的大部分端点默认应该受到保护，您可以将认证守卫注册为[全局守卫](/guards#binding-guards)，而不是在每个控制器上使用`@UseGuards()`装饰器，您可以简单地标记哪些路由应该是公开的。

首先，使用以下构造在任何模块中注册`AuthGuard`作为全局守卫（例如，在`AuthModule`中）：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: AuthGuard,
  },
],
```

有了这个，Nest将自动将`AuthGuard`绑定到所有端点。

现在，我们必须提供一个机制来声明路由为公开。为此，我们可以使用`SetMetadata`装饰器工厂函数创建一个自定义装饰器。

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

在上面的文件中，我们导出了两个常量。一个是我们的元数据键，名为`IS_PUBLIC_KEY`，另一个是我们的新装饰器，我们将称之为`Public`（您也可以称之为`SkipAuth`或`AllowAnon`，根据您的项目而定）。

现在我们已经有一个自定义的`@Public()`装饰器，我们可以使用它来装饰任何方法，如下所示：

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

最后，我们需要`AuthGuard`在找到`"isPublic"`元数据时返回`true`。为此，我们将使用`Reflector`类（了解更多[这里](/guards#putting-it-all-together)）。

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService, private reflector: Reflector) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      // 💡 看这个条件
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(token, {
        secret: jwtConstants.secret,
      });
      // 💡 我们在这里将payload分配给请求对象
      // 以便我们可以在路由处理程序中访问它
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

#### Passport集成

[Passport](https://github.com/jaredhanson/passport)是最受欢迎的node.js认证库，社区众所周知，并在许多生产应用程序中成功使用。使用`@nestjs/passport`模块将这个库与Nest应用程序集成是直接的。

要了解如何将Passport与NestJS集成，请查看这个[章节](/recipes/passport)。

#### 示例

您可以在[这里](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt)找到本章代码的完整版本。