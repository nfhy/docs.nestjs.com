### 加密和哈希

**加密**是编码信息的过程。这个过程将信息的原始表示形式，即明文，转换成另一种形式，称为密文。理想情况下，只有授权方能够将密文解密回明文并访问原始信息。加密本身并不阻止干扰，但否认了潜在拦截者对内容的理解。加密是一个双向函数；被加密的内容可以用正确的密钥解密。

**哈希**是将给定的密钥转换成另一个值的过程。哈希函数用于根据数学算法生成新值。一旦完成哈希，应该不可能从输出回到输入。

#### 加密

Node.js 提供了一个内置的[crypto模块](https://nodejs.org/api/crypto.html)，你可以使用它来加密和解密字符串、数字、缓冲区、流等。Nest 本身没有提供任何额外的包来避免引入不必要的抽象。

作为一个例子，让我们使用AES（高级加密系统）'aes-256-ctr'算法CTR加密模式。

```typescript
import { createCipheriv, randomBytes, scrypt } from 'crypto';
import { promisify } from 'util';

const iv = randomBytes(16);
const password = '用于生成密钥的密码';

// 密钥长度取决于算法。
// 在这种情况下，对于aes256，它是32字节。
const key = (await promisify(scrypt)(password, 'salt', 32)) as Buffer;
const cipher = createCipheriv('aes-256-ctr', key, iv);

const textToEncrypt = 'Nest';
const encryptedText = Buffer.concat([
  cipher.update(textToEncrypt),
  cipher.final(),
]);
```

现在解密`encryptedText`值：

```typescript
import { createDecipheriv } from 'crypto';

const decipher = createDecipheriv('aes-256-ctr', key, iv);
const decryptedText = Buffer.concat([
  decipher.update(encryptedText),
  decipher.final(),
]);
```

#### 哈希

对于哈希，我们推荐使用[bcrypt](https://www.npmjs.com/package/bcrypt)或[argon2](https://www.npmjs.com/package/argon2)包。Nest 本身没有提供任何额外的包装器来避免引入不必要的抽象（使学习曲线变短）。

作为一个例子，让我们使用`bcrypt`来哈希一个随机密码。

首先安装所需的包：

```shell
$ npm i bcrypt
$ npm i -D @types/bcrypt
```

安装完成后，你可以使用`hash`函数，如下所示：

```typescript
import * as bcrypt from 'bcrypt';

const saltOrRounds = 10;
const password = 'random_password';
const hash = await bcrypt.hash(password, saltOrRounds);
```

要生成盐，使用`genSalt`函数：

```typescript
const salt = await bcrypt.genSalt();
```

要比较/检查密码，使用`compare`函数：

```typescript
const isMatch = await bcrypt.compare(password, hash);
```

你可以在这里阅读更多关于可用函数的信息[here](https://www.npmjs.com/package/bcrypt)。