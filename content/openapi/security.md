### 安全性

要为特定操作定义应使用的安全性机制，请使用 `@ApiSecurity()` 装饰器。

```typescript
@ApiSecurity('basic')
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全性定义添加到您的基础文档中：

```typescript
const options = new DocumentBuilder().addSecurity('basic', {
  type: 'http',
  scheme: 'basic',
});
```

一些最受欢迎的认证技术是内置的（例如，`basic` 和 `bearer`），因此您不需要像上面所示那样手动定义安全性机制。

#### 基本认证

要启用基本认证，请使用 `@ApiBasicAuth()`。

```typescript
@ApiBasicAuth()
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全性定义添加到您的基础文档中：

```typescript
const options = new DocumentBuilder().addBasicAuth();
```

#### 持有者认证

要启用持有者认证，请使用 `@ApiBearerAuth()`。

```typescript
@ApiBearerAuth()
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全性定义添加到您的基础文档中：

```typescript
const options = new DocumentBuilder().addBearerAuth();
```

#### OAuth2 认证

要启用 OAuth2，请使用 `@ApiOAuth2()`。

```typescript
@ApiOAuth2(['pets:write'])
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全性定义添加到您的基础文档中：

```typescript
const options = new DocumentBuilder().addOAuth2();
```

#### Cookie 认证

要启用 Cookie 认证，请使用 `@ApiCookieAuth()`。

```typescript
@ApiCookieAuth()
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全性定义添加到您的基础文档中：

```typescript
const options = new DocumentBuilder().addCookieAuth('optional-session-id');
```