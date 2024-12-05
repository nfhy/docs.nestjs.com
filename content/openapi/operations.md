### 操作

在OpenAPI术语中，路径是端点（资源），例如`/users`或`/reports/summary`，这些是您的API公开的，而操作是用于操作这些路径的HTTP方法，如`GET`、`POST`或`DELETE`。

#### 标签

要将控制器附加到特定标签，使用`@ApiTags(...tags)`装饰器。

```typescript
@ApiTags('cats')
@Controller('cats')
export class CatsController {}
```

#### 头部

要定义作为请求一部分预期的自定义头部，使用`@ApiHeader()`。

```typescript
@ApiHeader({
  name: 'X-MyHeader',
  description: '自定义头部',
})
@Controller('cats')
export class CatsController {}
```

#### 响应

要定义自定义HTTP响应，请使用`@ApiResponse()`装饰器。

```typescript
@Post()
@ApiResponse({ status: 201, description: '记录已成功创建。'})
@ApiResponse({ status: 403, description: '禁止访问。'})
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Nest提供了一组继承自`@ApiResponse`装饰器的简短**API响应**装饰器：

- `@ApiOkResponse()`
- `@ApiCreatedResponse()`
- `@ApiAcceptedResponse()`
- `@ApiNoContentResponse()`
- `@ApiMovedPermanentlyResponse()`
- `@ApiFoundResponse()`
- `@ApiBadRequestResponse()`
- `@ApiUnauthorizedResponse()`
- `@ApiNotFoundResponse()`
- `@ApiForbiddenResponse()`
- `@ApiMethodNotAllowedResponse()`
- `@ApiNotAcceptableResponse()`
- `@ApiRequestTimeoutResponse()`
- `@ApiConflictResponse()`
- `@ApiPreconditionFailedResponse()`
- `@ApiTooManyRequestsResponse()`
- `@ApiGoneResponse()`
- `@ApiPayloadTooLargeResponse()`
- `@ApiUnsupportedMediaTypeResponse()`
- `@ApiUnprocessableEntityResponse()`
- `@ApiInternalServerErrorResponse()`
- `@ApiNotImplementedResponse()`
- `@ApiBadGatewayResponse()`
- `@ApiServiceUnavailableResponse()`
- `@ApiGatewayTimeoutResponse()`
- `@ApiDefaultResponse()`

```typescript
@Post()
@ApiCreatedResponse({ description: '记录已成功创建。'})
@ApiForbiddenResponse({ description: '禁止访问。'})
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

要为请求指定返回模型，我们必须创建一个类，并使用`@ApiProperty()`装饰器注解所有属性。

```typescript
export class Cat {
  @ApiProperty()
  id: number;

  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

然后，`Cat`模型可以与响应装饰器的`type`属性一起使用。

```typescript
@ApiTags('cats')
@Controller('cats')
export class CatsController {
  @Post()
  @ApiCreatedResponse({
    description: '记录已成功创建。',
    type: Cat,
  })
  async create(@Body() createCatDto: CreateCatDto): Promise<Cat> {
    return this.catsService.create(createCatDto);
  }
}
```

让我们打开浏览器并验证生成的`Cat`模型：

<figure><img src="/assets/swagger-response-type.png" /></figure>

#### 文件上传

您可以使用`@ApiBody`装饰器和`@ApiConsumes()`启用特定方法的文件上传。以下是一个使用[文件上传](/techniques/file-upload)技术的完整示例：

```typescript
@UseInterceptors(FileInterceptor('file'))
@ApiConsumes('multipart/form-data')
@ApiBody({
  description: '猫的列表',
  type: FileUploadDto,
})
uploadFile(@UploadedFile() file) {}
```

其中`FileUploadDto`定义如下：

```typescript
class FileUploadDto {
  @ApiProperty({ type: 'string', format: 'binary' })
  file: any;
}
```

要处理多个文件上传，您可以定义`FilesUploadDto`如下：

```typescript
class FilesUploadDto {
  @ApiProperty({ type: 'array', items: { type: 'string', format: 'binary' } })
  files: any[];
}
```

#### 扩展

要向请求添加扩展，请使用`@ApiExtension()`装饰器。扩展名称必须以`x-`为前缀。

```typescript
@ApiExtension('x-foo', { hello: 'world' })
```

#### 高级：通用`ApiResponse`

有了提供[原始定义](/openapi/types-and-parameters#raw-definitions)的能力，我们可以为Swagger UI定义通用模式。假设我们有以下DTO：

```ts
export class PaginatedDto<TData> {
  @ApiProperty()
  total: number;

  @ApiProperty()
  limit: number;

  @ApiProperty()
  offset: number;

  results: TData[];
}
```

我们跳过装饰`results`，因为我们稍后将为其提供原始定义。现在，让我们定义另一个DTO并命名为`CatDto`，如下所示：

```ts
export class CatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

有了这些，我们可以定义一个`PaginatedDto<CatDto>`响应，如下所示：

```ts
@ApiOkResponse({
  schema: {
    allOf: [
      { $ref: getSchemaPath(PaginatedDto) },
      {
        properties: {
          results: {
            type: 'array',
            items: { $ref: getSchemaPath(CatDto) },
          },
        },
      },
    ],
  },
})
async findAll(): Promise<PaginatedDto<CatDto>> {}
```

在这个例子中，我们指定响应将具有`PaginatedDto`的所有内容，并且`results`属性将是`Array<CatDto>`类型。

- `getSchemaPath()`函数返回OpenAPI规范文件中给定模型的OpenAPI Schema路径。
- `allOf`是OAS 3提供的一个概念，用于涵盖各种继承相关的用例。

最后，由于`PaginatedDto`没有被任何控制器直接引用，`SwaggerModule`将无法生成相应的模型定义。在这种情况下，我们必须将其作为[额外模型](/openapi/types-and-parameters#extra-models)添加。例如，我们可以使用`@ApiExtraModels()`装饰器在控制器级别上，如下所示：

```ts
@Controller('cats')
@ApiExtraModels(PaginatedDto)
export class CatsController {}
```

如果您现在运行Swagger，此特定端点的生成`swagger.json`应该具有以下响应定义：

```json
"responses": {
  "200": {
    "description": "",
    "content": {
      "application/json": {
        "schema": {
          "allOf": [
            {
              "$ref": "#/components/schemas/PaginatedDto"
            },
            {
              "properties": {
                "results": {
                  "$ref": "#/components/schemas/CatDto"
                }
              }
            }
          ]
        }
      }
    }
  }
}
```

为了使其可重用，我们可以为`PaginatedDto`创建一个自定义装饰器，如下所示：

```ts
export const ApiPaginatedResponse = <TModel extends Type<any>>(
  model: TModel,
) => {
  return applyDecorators(
    ApiExtraModels(PaginatedDto, model),
    ApiOkResponse({
      schema: {
        allOf: [
          { $ref: getSchemaPath(PaginatedDto) },
          {
            properties: {
              results: {
                type: 'array',
                items: { $ref: getSchemaPath(model) },
              },
            },
          },
        ],
      },
    }),
  );
};
```

> 信息提示 `Type<any>`接口和`applyDecorators`函数从`@nestjs/common`包导入。

为确保`SwaggerModule`将为我们的模型生成定义，我们必须像之前对`PaginatedDto`那样将其作为额外模型添加。

有了这些，我们可以使用自定义的`@ApiPaginatedResponse()`装饰器在我们的端点上：

```ts
@ApiPaginatedResponse(CatDto)
async findAll(): Promise<PaginatedDto<CatDto>> {}
```

对于客户端生成工具，这种方法在生成客户端时`PaginatedResponse<TModel>`的生成方式上存在歧义。以下是一个客户端生成器结果的示例，针对上述`GET /`端点。

```typescript
// Angular
findAll(): Observable<{ total: number, limit: number, offset: number, results: CatDto[] }>
```

如您所见，**返回类型**在这里是模糊的。为了解决这个问题，您可以为`ApiPaginatedResponse`的`schema`添加一个`title`属性：

```typescript
export const ApiPaginatedResponse = <TModel extends Type<any>>(model: TModel) => {
  return applyDecorators(
    ApiOkResponse({
      schema: {
        title: `PaginatedResponseOf${model.name}`,
        allOf: [
          // ...
        ],
      },
    }),
  );
};
```

现在客户端生成器工具的结果将变为：

```ts
// Angular
findAll(): Observable<PaginatedResponseOfCatDto>
```