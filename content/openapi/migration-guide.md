### 迁移指南

如果您目前正在使用 `@nestjs/swagger@3.*`，请留意版本 4.0 中的以下破坏性/API变更。

#### 破坏性变更

以下装饰器已更改/重命名：

- `@ApiModelProperty` 现在是 `@ApiProperty`
- `@ApiModelPropertyOptional` 现在是 `@ApiPropertyOptional`
- `@ApiResponseModelProperty` 现在是 `@ApiResponseProperty`
- `@ApiImplicitQuery` 现在是 `@ApiQuery`
- `@ApiImplicitParam` 现在是 `@ApiParam`
- `@ApiImplicitBody` 现在是 `@ApiBody`
- `@ApiImplicitHeader` 现在是 `@ApiHeader`
- `@ApiOperation({{ '{' }} title: 'test' {{ '}' }})` 现在是 `@ApiOperation({{ '{' }} summary: 'test' {{ '}' }})`
- `@ApiUseTags` 现在是 `@ApiTags`

`DocumentBuilder` 破坏性变更（更新的方法签名）：

- `addTag`
- `addBearerAuth`
- `addOAuth2`
- `setContactEmail` 现在是 `setContact`
- `setHost` 已被移除
- `setSchemes` 已被移除（使用 `addServer` 替代，例如 `addServer('http://')`）

#### 新增方法

以下方法已添加：

- `addServer`
- `addApiKey`
- `addBasicAuth`
- `addSecurity`
- `addSecurityRequirements`