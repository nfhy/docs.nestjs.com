### 任务调度

任务调度允许您安排任意代码（方法/函数）在固定日期/时间执行，以重复的间隔执行，或在指定间隔后执行一次。在Linux世界中，这通常由操作系统级别的cron包处理。对于Node.js应用，有几个包模拟了cron类似的功能。Nest提供了`@nestjs/schedule`包，它与流行的Node.js [cron](https://github.com/kelektiv/node-cron)包集成。我们将在当前章节中介绍这个包。

#### 安装

要开始使用它，我们首先安装所需的依赖项。

```bash
$ npm install --save @nestjs/schedule
```

要激活作业调度，请将`ScheduleModule`导入到根`AppModule`中，并运行`forRoot()`静态方法，如下所示：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot()
  ],
})
export class AppModule {}
```

`.forRoot()`调用初始化调度器并注册应用程序中的任何声明式[cron作业](techniques/task-scheduling#declarative-cron-jobs)、[超时](techniques/task-scheduling#declarative-timeouts)和[间隔](techniques/task-scheduling#declarative-intervals)。注册发生在`onApplicationBootstrap`生命周期钩子发生时，确保所有模块都已加载并声明了任何计划作业。

#### 声明式cron作业

cron作业安排一个任意函数（方法调用）自动运行。cron作业可以运行：

- 一次，在指定的日期/时间。
- 重复运行；重复作业可以在指定间隔内的指定瞬间运行（例如，每小时一次，每周一次，每5分钟一次）

使用`@Cron()`装饰器在包含要执行代码的方法定义之前声明cron作业，如下所示：

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron('45 * * * * *')
  handleCron() {
    this.logger.debug('Called when the current second is 45');
  }
}
```

在这个例子中，`handleCron()`方法将在当前秒为`45`时每次被调用。换句话说，该方法将每分钟运行一次，在45秒标记时运行。

`@Cron()`装饰器支持以下标准[cron模式](http://crontab.org/)：

- 星号（例如`*`）
- 范围（例如`1-3,5`）
- 步骤（例如`*/2`）

在上面的例子中，我们向装饰器传递了`45 * * * * *`。以下键显示了cron模式字符串中每个位置的解释：

<pre class="language-javascript"><code class="language-javascript">
* * * * * *
| | | | | |
| | | | | day of week
| | | | months
| | | day of month
| | hours
| minutes
seconds (optional)
</code></pre>

一些cron模式示例如下：

<table>
  <tbody>
    <tr>
      <td><code>* * * * * *</code></td>
      <td>每秒</td>
    </tr>
    <tr>
      <td><code>45 * * * * *</code></td>
      <td>每分钟，在第45秒</td>
    </tr>
    <tr>
      <td><code>0 10 * * * *</code></td>
      <td>每小时，在第10分钟开始时</td>
    </tr>
    <tr>
      <td><code>0 */30 9-17 * * *</code></td>
      <td>每天上午9点到下午5点之间，每30分钟</td>
    </tr>
    <tr>
      <td><code>0 30 11 * * 1-5</code></td>
      <td>周一至周五，上午11:30</td>
    </tr>
  </tbody>
</table>

`@nestjs/schedule`包提供了一个方便的枚举，包含常用的cron模式。您可以按如下方式使用此枚举：

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron(CronExpression.EVERY_30_SECONDS)
  handleCron() {
    this.logger.debug('Called every 30 seconds');
  }
}
```

在这个例子中，`handleCron()`方法将每`30`秒被调用一次。如果发生异常，它将被记录到控制台，因为每个用`@Cron()`注解的方法都会自动包装在一个`try-catch`块中。

或者，您可以向`@Cron()`装饰器提供一个JavaScript `Date`对象。这样做会导致作业在指定的日期精确执行一次。

> 信息提示 使用JavaScript日期算术相对于当前日期安排作业。例如，`@Cron(new Date(Date.now() + 10 * 1000))`安排作业在应用启动后10秒运行。

此外，您可以将额外的选项作为第二个参数提供给`@Cron()`装饰器。

<table>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>有助于在声明后访问和控制cron作业。</td>
    </tr>
    <tr>
      <td><code>timeZone</code></td>
      <td>指定执行的时区。这将修改相对于您时区的实际时间。如果时区无效，将抛出错误。您可以在<a href="http://momentjs.com/timezone/">Moment Timezone</a>网站上查看所有可用的时区。</td>
    </tr>
    <tr>
      <td><code>utcOffset</code></td>
      <td>这允许您指定时区的偏移量，而不是使用<code>timeZone</code>参数。</td>
    </tr>
    <tr>
      <td><code>disabled</code></td>
      <td>这表示作业是否根本不会被执行。</td>
    </tr>
  </tbody>
</table>

```typescript
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class NotificationService {
  @Cron('* * 0 * * *', {
    name: 'notifications',
    timeZone: 'Europe/Paris',
  })
  triggerNotifications() {}
}
```

您可以在声明后访问和控制cron作业，或者使用[动态调度模块API](/techniques/task-scheduling#dynamic-schedule-module-api)动态创建cron作业（其中cron模式在运行时定义）。要通过API访问声明式cron作业，您必须通过在装饰器的第二个可选选项对象中传递`name`属性来将作业与名称关联。

#### 声明式间隔

要声明一个方法应该在（重复的）指定间隔运行，请在方法定义前加上`@Interval()`装饰器。将间隔值作为毫秒数传递给装饰器，如下所示：

```typescript
@Interval(10000)
handleInterval() {
  this.logger.debug('Called every 10 seconds');
}
```

> 信息提示 此机制在底层使用JavaScript `setInterval()`函数。您也可以使用cron作业来安排重复作业。

如果您想通过[动态API](/techniques/task-scheduling#dynamic-schedule-module-api)从声明类外部控制您的声明式间隔，请使用以下构造将间隔与名称关联：

```typescript
@Interval('notifications', 2500)
handleInterval() {}
```

如果发生异常，它将被记录到控制台，因为每个用`@Interval()`注解的方法都会自动包装在一个`try-catch`块中。

[动态API](/techniques/task-scheduling#dynamic-intervals)还允许**创建**动态间隔，其中间隔的属性在运行时定义，以及**列出和删除**它们。

#### 声明式超时

要声明一个方法应该在指定的超时后（一次）运行，请在方法定义前加上`@Timeout()`装饰器。从应用启动传递相对时间偏移（以毫秒为单位）给装饰器，如下所示：

```typescript
@Timeout(5000)
handleTimeout() {
  this.logger.debug('Called once after 5 seconds');
}
```

> 信息提示 此机制在底层使用JavaScript `setTimeout()`函数。

如果发生异常，它将被记录到控制台，因为每个用`@Timeout()`注解的方法都会自动包装在一个`try-catch`块中。

如果您想通过[动态API](/techniques/task-scheduling#dynamic-schedule-module-api)从声明类外部控制您的声明式超时，请使用以下构造将超时与名称关联：

```typescript
@Timeout('notifications', 2500)
handleTimeout() {}
```

[动态API](/techniques/task-scheduling#dynamic-timeouts)还允许**创建**动态超时，其中超时的属性在运行时定义，以及**列出和删除**它们。

#### 动态调度模块API

`@nestjs/schedule`模块提供了一个动态API，允许管理声明式的[cron作业](techniques/task-scheduling#declarative-cron-jobs)、[超时](techniques/task-scheduling#declarative-timeouts)和[间隔](techniques/task-scheduling#declarative-intervals)。该API还允许创建和管理**动态**的cron作业、超时和间隔，其中属性在运行时定义。

#### 动态cron作业

使用`SchedulerRegistry` API从代码的任何位置通过名称获取对`CronJob`实例的引用。首先，使用标准构造函数注入注入`SchedulerRegistry`：

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

> 信息提示 从`@nestjs/schedule`包导入`SchedulerRegistry`。

然后在类中使用它，如下所示。假设一个cron作业是用以下声明创建的：

```typescript
@Cron('* * 8 * * *', {
  name: 'notifications',
})
triggerNotifications() {}
```

使用以下方式访问此作业：

```typescript
const job = this.schedulerRegistry.getCronJob('notifications');

job.stop();
console.log(job.lastDate());
```

`getCronJob()`方法返回命名的cron作业。返回的`CronJob`对象具有以下方法：

- `stop()` - 停止计划运行的作业。
- `start()` - 重新启动已停止的作业。
- `setTime(time: CronTime)` - 停止作业，为其设置新的时间，然后启动它
- `lastDate()` - 返回上次作业执行日期的`DateTime`表示。
- `nextDate()` - 返回计划下一次作业执行的日期的`DateTime`表示。
- `nextDates(count: number)` - 提供一个数组（大小`count`）的`DateTime`表示，用于下一次触发作业执行的日期集合。`count`默认为0，返回空数组。

> 信息提示 使用`DateTime`对象上的`toJSDate()`将它们渲染为JavaScript Date等效的此DateTime。

**创建**一个新的cron作业动态使用`SchedulerRegistry#addCronJob`方法，如下所示：

```typescript
addCronJob(name: string, seconds: string) {
  const job = new CronJob(`${seconds} * * * * *`, () => {
    this.logger.warn(`time (${seconds}) for job ${name} to run!`);
  });

  this.schedulerRegistry.addCronJob(name, job);
  job.start();

  this.logger.warn(
    `job ${name} added for each minute at ${seconds} seconds!`,
  );
}
```

在此代码中，我们使用`cron`包中的`CronJob`对象创建cron作业。`CronJob`构造函数采用cron模式（就像`@Cron()`[装饰器](techniques/task-scheduling#declarative-cron-jobs)）作为其第一个参数，以及当cron计时器触发时要执行的回调作为其第二个参数。`SchedulerRegistry#addCronJob`方法接受两个参数：`CronJob`的名称和`CronJob`对象本身。

> 警告 提示 在访问之前记住注入`SchedulerRegistry`。从`cron`包导入`CronJob`。

**删除**使用`SchedulerRegistry#deleteCronJob`方法的命名cron作业，如下所示：

```typescript
deleteCron(name: string) {
  this.schedulerRegistry.deleteCronJob(name);
  this.logger.warn(`job ${name} deleted!`);
}
```

**列出**所有cron作业使用`SchedulerRegistry#getCronJobs`方法如下：

```typescript
getCrons() {
  const jobs = this.schedulerRegistry.getCronJobs();
  jobs.forEach((value, key, map) => {
    let next;
    try {
      next = value.nextDate().toJSDate();
    } catch (e) {
      next = 'error: next fire date is in the past!';
    }
    this.logger.log(`job: ${key} -> next: ${next}`);
  });
}
```

`getCronJobs()`方法返回一个`map`。在此代码中，我们迭代map并尝试访问每个`CronJob`的`nextDate()`方法。在`CronJob` API中，如果作业已经触发并且没有未来的触发日期，它会抛出异常。

#### 动态间隔

使用`SchedulerRegistry#getInterval`方法获取对间隔的引用。如上所述，使用标准构造函数注入注入`SchedulerRegistry`：

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

并如下使用它：

```typescript
const interval = this.schedulerRegistry.getInterval('notifications');
clearInterval(interval);
```

**创建**一个新的间隔动态使用`SchedulerRegistry#addInterval`方法，如下所示：

```typescript
addInterval(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Interval ${name} executing at time (${milliseconds})!`);
  };

  const interval = setInterval(callback, milliseconds);
  this.schedulerRegistry.addInterval(name, interval);
}
```

在此代码中，我们创建一个标准JavaScript间隔，然后将其传递给`SchedulerRegistry#addInterval`方法。该方法接受两个参数：间隔的名称和间隔本身。

**删除**使用`SchedulerRegistry#deleteInterval`方法的命名间隔，如下所示：

```typescript
deleteInterval(name: string) {
  this.schedulerRegistry.deleteInterval(name);
  this.logger.warn(`Interval ${name} deleted!`);
}
```

**列出**所有间隔使用`SchedulerRegistry#getIntervals`方法如下：

```typescript
getIntervals() {
  const intervals = this.schedulerRegistry.getIntervals();
  intervals.forEach(key => this.logger.log(`Interval: ${key}`));
}
```

#### 动态超时

使用`SchedulerRegistry#getTimeout`方法获取对超时的引用。如上所述，使用标准构造函数注入注入`SchedulerRegistry`：

```typescript
constructor(private readonly schedulerRegistry: SchedulerRegistry) {}
```

并如下使用它：

```typescript
const timeout = this.schedulerRegistry.getTimeout('notifications');
clearTimeout(timeout);
```

**创建**一个新的超时动态使用`SchedulerRegistry#addTimeout`方法，如下所示：

```typescript
addTimeout(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Timeout ${name} executing after (${milliseconds})!`);
  };

  const timeout = setTimeout(callback, milliseconds);
  this.schedulerRegistry.addTimeout(name, timeout);
}
```

在此代码中，我们创建一个标准JavaScript超时，然后将其传递给`SchedulerRegistry#addTimeout`方法。该方法接受两个参数：超时的名称和超时本身。

**删除**使用`SchedulerRegistry#deleteTimeout`方法的命名超时，如下所示：

```typescript
deleteTimeout(name: string) {
  this.schedulerRegistry.deleteTimeout(name);
  this.logger.warn(`Timeout ${name} deleted!`);
}
```

**列出**所有超时使用`SchedulerRegistry#getTimeouts`方法如下：

```typescript
getTimeouts() {
  const timeouts = this.schedulerRegistry.getTimeouts();
  timeouts.forEach(key => this.logger.log(`Timeout: ${key}`));
}
```

#### 示例

一个工作示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/27-scheduling)找到。