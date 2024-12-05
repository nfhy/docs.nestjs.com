### 事件

[事件发射器](https://www.npmjs.com/package/@nestjs/event-emitter) 包 (`@nestjs/event-emitter`) 提供了一个简单的观察者实现，允许您订阅并监听应用程序中发生的各种事件。事件作为解耦应用程序各个部分的绝佳方式，因为一个事件可以有多个不相互依赖的监听器。

`EventEmitterModule` 内部使用 [eventemitter2](https://github.com/EventEmitter2/EventEmitter2) 包。

#### 开始使用

首先安装所需的包：

```shell
$ npm i --save @nestjs/event-emitter
```

安装完成后，将 `EventEmitterModule` 导入到根 `AppModule` 并运行 `forRoot()` 静态方法，如下所示：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot()
  ],
})
export class AppModule {}
```

`.forRoot()` 调用初始化事件发射器并注册应用程序中的任何声明性事件监听器。注册发生在 `onApplicationBootstrap` 生命周期钩子发生时，确保所有模块都已加载并声明了任何计划任务。

要配置底层的 `EventEmitter` 实例，请将配置对象传递给 `.forRoot()` 方法，如下：

```typescript
EventEmitterModule.forRoot({
  // 将此设置为 `true` 以使用通配符
  wildcard: false,
  // 用于分隔命名空间的分隔符
  delimiter: '.',
  // 将此设置为 `true` 如果您想发出 newListener 事件
  newListener: false,
  // 将此设置为 `true` 如果您想发出 removeListener 事件
  removeListener: false,
  // 可以分配给事件的最大监听器数量
  maxListeners: 10,
  // 当分配的监听器超过最大数量时显示事件名称在内存泄漏消息中
  verboseMemoryLeak: false,
  // 如果一个错误事件被发出并且没有监听器，禁用抛出 uncaughtException
  ignoreErrors: false,
});
```

#### 派发事件

要派发（即，触发）一个事件，首先使用标准构造函数注入注入 `EventEmitter2`：

```typescript
constructor(private eventEmitter: EventEmitter2) {}
```

> 信息 **提示** 从 `@nestjs/event-emitter` 包中导入 `EventEmitter2`。

然后在类中如下使用：

```typescript
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({
    orderId: 1,
    payload: {},
  }),
);
```

#### 监听事件

要声明一个事件监听器，请用 `@OnEvent()` 装饰器装饰一个方法，该方法定义包含要执行的代码，如下：

```typescript
@OnEvent('order.created')
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // 处理和处理 "OrderCreatedEvent" 事件
}
```

> 警告 **警告** 事件订阅者不能是请求范围的。

第一个参数可以是 `string` 或 `symbol` 用于简单的事件发射器，以及 `string | symbol | Array<string | symbol>` 在通配符发射器的情况下。

第二个参数（可选）是监听器选项对象，如下：

```typescript
export type OnEventOptions = OnOptions & {
  /**
   * 如果为 "true"，则将给定的监听器添加到监听器数组的前面（而不是后面）。
   *
   * @see https://github.com/EventEmitter2/EventEmitter2#emitterprependlistenerevent-listener-options
   *
   * @default false
   */
  prependListener?: boolean;

  /**
   * 如果为 "true"，则在处理事件时 onEvent 回调不会抛出错误。否则，如果为 "false" 则会抛出错误。
   *
   * @default true
   */
  suppressErrors?: boolean;
};
```

> 信息 **提示** 从 [`eventemitter2`](https://github.com/EventEmitter2/EventEmitter2#emitteronevent-listener-options-objectboolean) 了解更多关于 `OnOptions` 选项对象的信息。

```typescript
@OnEvent('order.created', { async: true })
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // 处理和处理 "OrderCreatedEvent" 事件
}
```

要使用命名空间/通配符，将 `wildcard` 选项传递到 `EventEmitterModule#forRoot()` 方法。当命名空间/通配符功能启用时，事件可以是字符串（`foo.bar`）通过分隔符分隔或数组（`['foo', 'bar']`）。分隔符也是可配置的配置属性（`delimiter`）。使用命名空间功能启用时，您可以使用通配符订阅事件：

```typescript
@OnEvent('order.*')
handleOrderEvents(payload: OrderCreatedEvent | OrderRemovedEvent | OrderUpdatedEvent) {
  // 处理和处理事件
}
```

请注意，这样的通配符只适用于一个区块。参数 `order.*` 将匹配例如 `order.created` 和 `order.shipped` 事件，但不匹配 `order.delayed.out_of_stock`。为了监听这样的事件，请使用 `多级通配符` 模式（即，`**`），在 `EventEmitter2` [文档](https://github.com/EventEmitter2/EventEmitter2#multi-level-wildcards)中描述。

使用这种模式，例如，您可以创建一个事件监听器来捕获所有事件。

```typescript
@OnEvent('**')
handleEverything(payload: any) {
  // 处理和处理事件
}
```

> 信息 **提示** `EventEmitter2` 类提供了几个有用的方法来与事件交互，如 `waitFor` 和 `onAny`。您可以在 [这里](https://github.com/EventEmitter2/EventEmitter2) 了解更多信息。

#### 防止事件丢失

在 `onApplicationBootstrap` 生命周期钩子之前或期间触发的事件（例如，来自模块构造函数或 `onModuleInit` 方法的事件）可能会丢失，因为 `EventSubscribersLoader` 可能尚未完成设置监听器。

为了避免这个问题，您可以使用 `EventEmitterReadinessWatcher` 的 `waitUntilReady` 方法，该方法返回一个承诺，一旦所有监听器都已注册，该承诺就会解决。这个方法可以在模块的 `onApplicationBootstrap` 生命周期钩子中调用，以确保所有事件都被正确捕获。

```typescript
await this.eventEmitterReadinessWatcher.waitUntilReady();
await this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({ orderId: 1, payload: {} }),
);
```

> 注意 **注意** 这只对在 `onApplicationBootstrap` 生命周期钩子完成之前发出的事件是必要的。

#### 示例

一个工作示例可以在 [这里](https://github.com/nestjs/nest/tree/master/sample/30-event-emitter) 找到。