### Necord

Necord 是一个强大的模块，它简化了 [Discord](https://discord.com) 机器人的创建，允许与你的 NestJS 应用程序无缝集成。

> **注意**：Necord 是第三方包，不是由 NestJS 核心团队官方维护的。如果你遇到任何问题，请在 [官方仓库](https://github.com/necordjs/necord) 中报告。

#### 安装

要开始使用，你需要安装 Necord 及其依赖项 [`Discord.js`](https://discord.js.org)。

```bash
$ npm install necord discord.js
```

#### 使用

在你的项目中使用 Necord，导入 `NecordModule` 并用必要的选项进行配置。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { NecordModule } from 'necord';
import { IntentsBitField } from 'discord.js';
import { AppService } from './app.service';

@Module({
  imports: [
    NecordModule.forRoot({
      token: process.env.DISCORD_TOKEN,
      intents: [IntentsBitField.Guilds],
      development: [process.env.DISCORD_DEVELOPMENT_GUILD_ID],
    }),
  ],
  providers: [AppService],
})
export class AppModule {}
```

> **提示**：你可以在这里找到可用意图的完整列表 [这里](https://discord.com/developers/docs/topics/gateway#gateway-intents)。

通过这种设置，你可以将 `AppService` 注入到你的提供者中，以轻松注册命令、事件等。

```typescript
@@filename(app.service)
import { Injectable, Logger } from '@nestjs/common';
import { Context, On, Once, ContextOf } from 'necord';
import { Client } from 'discord.js';

@Injectable()
export class AppService {
  private readonly logger = new Logger(AppService.name);

  @Once('ready')
  public onReady(@Context() [client]: ContextOf<'ready'>) {
    this.logger.log(`Bot logged in as ${client.user.username}`);
  }

  @On('warn')
  public onWarn(@Context() [message]: ContextOf<'warn'>) {
    this.logger.warn(message);
  }
}
```

##### 理解上下文

你可能已经注意到了上述示例中的 `@Context` 装饰器。这个装饰器将事件上下文注入到你的方法中，允许你访问各种事件特定的数据。由于有多种类型的事件，上下文类型是使用 `ContextOf<type: string>` 类型推断的。你可以轻松地使用 `@Context()` 装饰器访问上下文变量，它用与事件相关的参数数组填充变量。

#### 文本命令

> **警告**：文本命令依赖于消息内容，这将被废弃用于验证的机器人和拥有超过 100 个服务器的应用程序。这意味着如果你的机器人无法访问消息内容，文本命令将无法工作。阅读更多关于这个变化的信息 [这里](https://support-dev.discord.com/hc/en-us/articles/4404772028055-Message-Content-Access-Deprecation-for-Verified-Bots)。

以下是如何使用 `@TextCommand` 装饰器创建简单的命令处理器。

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, TextCommand, TextCommandContext, Arguments } from 'necord';

@Injectable()
export class AppCommands {
  @TextCommand({
    name: 'ping',
    description: 'Responds with pong!',
  })
  public onPing(
    @Context() [message]: TextCommandContext,
    @Arguments() args: string[],
  ) {
    return message.reply('pong!');
  }
}
```

#### 应用程序命令

应用程序命令为用户提供了一种原生的方式在 Discord 客户端内与你的应用程序互动。有三种类型的应用程序命令可以通过不同的界面访问：聊天输入、消息上下文菜单（通过右键单击消息访问）和用户上下文菜单（通过右键单击用户访问）。

<figure><img class="illustrative-image" src="https://i.imgur.com/4EmG8G8.png" /></figure>

#### 斜杠命令

斜杠命令是与用户互动的一种结构化方式。它们允许你创建具有精确参数和选项的命令，显著增强用户体验。

要使用 Necord 定义斜杠命令，你可以使用 `SlashCommand` 装饰器。

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, SlashCommand, SlashCommandContext } from 'necord';

@Injectable()
export class AppCommands {
  @SlashCommand({
    name: 'ping',
    description: 'Responds with pong!',
  })
  public async onPing(@Context() [interaction]: SlashCommandContext) {
    return interaction.reply({ content: 'Pong!' });
  }
}
```

> **提示**：当你的机器人客户端登录时，它会自动注册所有定义的命令。注意，全局命令会缓存最多一个小时。为了避免全局缓存的问题，使用 Necord 模块的 `development` 参数，它将命令的可见性限制在单个公会中。

##### 选项

你可以使用选项装饰器为你的斜杠命令定义参数。我们为此目的创建一个 `TextDto` 类：

```typescript
@@filename(text.dto)
import { StringOption } from 'necord';

export class TextDto {
  @StringOption({
    name: 'text',
    description: 'Input your text here',
    required: true,
  })
  text: string;
}
```

然后你可以在 `AppCommands` 类中使用这个 DTO：

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, SlashCommand, Options, SlashCommandContext } from 'necord';
import { TextDto } from './length.dto';

@Injectable()
export class AppCommands {
  @SlashCommand({
    name: 'length',
    description: 'Calculate the length of your text',
  })
  public async onLength(
    @Context() [interaction]: SlashCommandContext,
    @Options() { text }: TextDto,
  ) {
    return interaction.reply({
      content: `The length of your text is: ${text.length}`,
    });
  }
}
```

对于内置选项装饰器的完整列表，请查看 [这个文档](https://necord.org/interactions/slash-commands#options)。

##### 自动补全

要为你的斜杠命令实现自动补全功能，你需要创建一个拦截器。这个拦截器将处理用户在自动补全字段中输入时的请求。

```typescript
@@filename(cats-autocomplete.interceptor)
import { Injectable } from '@nestjs/common';
import { AutocompleteInteraction } from 'discord.js';
import { AutocompleteInterceptor } from 'necord';

@Injectable()
class CatsAutocompleteInterceptor extends AutocompleteInterceptor {
  public transformOptions(interaction: AutocompleteInteraction) {
    const focused = interaction.options.getFocused(true);
    let choices: string[];

    if (focused.name === 'cat') {
      choices = ['Siamese', 'Persian', 'Maine Coon'];
    }

    return interaction.respond(
      choices
        .filter((choice) => choice.startsWith(focused.value.toString()))
        .map((choice) => ({ name: choice, value: choice })),
    );
  }
}
```

你还需要在你的选项类上标记 `autocomplete: true`：

```typescript
@@filename(cat.dto)
import { StringOption } from 'necord';

export class CatDto {
  @StringOption({
    name: 'cat',
    description: 'Choose a cat breed',
    autocomplete: true,
    required: true,
  })
  cat: string;
}
```

最后，将拦截器应用到你的斜杠命令：

```typescript
@@filename(cats.commands)
import { Injectable, UseInterceptors } from '@nestjs/common';
import { Context, SlashCommand, Options, SlashCommandContext } from 'necord';
import { CatDto } from '/cat.dto';
import { CatsAutocompleteInterceptor } from './cats-autocomplete.interceptor';

@Injectable()
export class CatsCommands {
  @UseInterceptors(CatsAutocompleteInterceptor)
  @SlashCommand({
    name: 'cat',
    description: 'Retrieve information about a specific cat breed',
  })
  public async onSearch(
    @Context() [interaction]: SlashCommandContext,
    @Options() { cat }: CatDto,
  ) {
    return interaction.reply({
      content: `I found information on the breed of ${cat} cat!`,
    });
  }
}
```

#### 用户上下文菜单

用户命令出现在右键单击（或点击）用户时出现的上下文菜单上。这些命令提供直接针对用户的快速操作。

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, UserCommand, UserCommandContext, TargetUser } from 'necord';
import { User } from 'discord.js';

@Injectable()
export class AppCommands {
  @UserCommand({ name: 'Get avatar' })
  public async getUserAvatar(
    @Context() [interaction]: UserCommandContext,
    @TargetUser() user: User,
  ) {
    return interaction.reply({
      embeds: [
        new MessageEmbed()
          .setTitle(`Avatar of ${user.username}`)
          .setImage(user.displayAvatarURL({ size: 4096, dynamic: true })),
      ],
    });
  }
}
```

#### 消息上下文菜单

消息命令在右键单击消息时出现在上下文菜单中，允许与这些消息相关的快速操作。

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, MessageCommand, MessageCommandContext, TargetMessage } from 'necord';
import { Message } from 'discord.js';

@Injectable()
export class AppCommands {
  @MessageCommand({ name: 'Copy Message' })
  public async copyMessage(
    @Context() [interaction]: MessageCommandContext,
    @TargetMessage() message: Message,
  ) {
    return interaction.reply({ content: message.content });
  }
}
```

#### 按钮

[按钮](https://discord.com/developers/docs/interactions/message-components#buttons) 是可以包含在消息中的交互式元素。当点击时，它们会向你的应用程序发送 [交互](https://discord.com/developers/docs/interactions/receiving-and-responding#interaction-object)。

```typescript
@@filename(app.components)
import { Injectable } from '@nestjs/common';
import { Context, Button, ButtonContext } from 'necord';

@Injectable()
export class AppComponents {
  @Button('BUTTON')
  public onButtonClick(@Context() [interaction]: ButtonContext) {
    return interaction.reply({ content: 'Button clicked!' });
  }
}
```

#### 选择菜单

[选择菜单](https://discord.com/developers/docs/interactions/message-components#select-menus) 是另一种出现在消息中的交互式组件。它们为用户提供了类似下拉菜单的 UI，供用户选择选项。

```typescript
@@filename(app.components)
import { Injectable } from '@nestjs/common';
import { Context, StringSelect, StringSelectContext, SelectedStrings } from 'necord';

@Injectable()
export class AppComponents {
  @StringSelect('SELECT_MENU')
  public onSelectMenu(
    @Context() [interaction]: StringSelectContext,
    @SelectedStrings() values: string[],
  ) {
    return interaction.reply({ content: `You selected: ${values.join(', ')}` });
  }
}
```

对于内置选择菜单组件的完整列表，请访问 [这个链接](https://necord.org/interactions/message-components#select-menu)。

#### 模态框

模态框是弹出表单，允许用户提交格式化输入。以下是如何使用 Necord 创建和处理模态框：

```typescript
@@filename(app.modals)
import { Injectable } from '@nestjs/common';
import { Context, Modal, ModalContext } from 'necord';

@Injectable()
export class AppModals {
  @Modal('pizza')
  public onModal(@Context() [interaction]: ModalContext) {
    return interaction.reply({
      content: `Your fav pizza : ${interaction.fields.getTextInputValue('pizza')}`
    });
  }
}
```

#### 更多信息

访问 [Necord](https://necord.org) 网站了解更多信息。