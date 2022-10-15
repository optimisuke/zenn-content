---
title: "TSyringe で DI Container と Singleton"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript, tsyringe, di]
published: true
---

# はじめに

TypeScript で DI Container 作りたくなったので、調査して試してみた。
前回書いた記事（[TypeScript で依存性逆転の原則と DI とテスト](https://zenn.dev/optimisuke/articles/d13c5e206f5369)）では、DI Container について書かなかったので、こっちで補完出来たらいいかなと思う。

まず、DI Container の概念については、[DI・DI コンテナ、ちゃんと理解出来てる・・？ - Qiita](https://qiita.com/ritukiii/items/de30b2d944109521298f)とか、[ドメイン駆動設計入門 ボトムアップでわかる！ドメイン駆動設計の基本（成瀬 允宣）｜翔泳社の本](https://www.shoeisha.co.jp/book/detail/9784798150727)を読んでもらったら良いかと思う。
自分の理解も書いておくと、ただの DI（特に Constructor Injection）だと、コードのトップで new しまくらないといけないのが微妙だから、依存関係はできるだけメインの処理とは別のところに設定ファイルっぽくコンテナ（Docker とは無関係）としてまとめちゃいなよって概念なのかなと思ってる。（指摘あればウェルカムです！）

TypeScript で DI Container する方法を調べると、[InversifyJS](https://github.com/inversify/InversifyJS) と [TSyringe](https://github.com/microsoft/tsyringe#inject)が有名みたい。
自分で作ろうと思うと、以下の記事を参考にしたらよさそう。ただ、TypeScript は JavaScript にトランスパイルするって制約があるため、TypeScript レベルを上げてからでないと難しそう。
[DI/DI コンテナとは？TypeScript での軽量 DI 実装まで - Qiita](https://qiita.com/tak001/items/83bdb140e2e0df13df09)

シンプルにやりたかったので、まずは[TSyringe](https://github.com/microsoft/tsyringe#inject)を試してみた。
いい感じの情報ないかなと探してると、以下の記事がやってることシンプルでわかりやすかった。
[TypeScript の DI と Tsyringe について](https://zenn.dev/chida/articles/1f7df8f2beb6b6)

ただ、トップのファイルで register してるから、それって new してるのとほとんどおんなじやんってなった。
なので、いい感じのファイル構成で試してる人いないか探してみた。すると下記サイトを見つけたので、参考にした。
[Using tsyringe for dependency injection without using the class syntax - DEV Community 👩‍💻👨‍💻](https://dev.to/leehambley/using-tsyringe-for-dependency-injection-without-using-the-class-syntax-29h7)

他にも調べてると、TSyringe とか InversifyJS を使うと、楽にシングルトン作れることも分かった。
この記事が超絶わかりやすかった。InversifyJS で書かれてたので TSyringe でも試してみることにした。
[Node で書く時 TypeScript で DI したい | CYOKODOG](https://www.cyokodog.net/blog/inversify-js/)

以下では、下記 2 点を書く。
TSyringe とか DI Container を使おうとしている人向けに、上記参照先の補完情報を提供できればと思う。

1. TSyringe で DI Container
2. TSyringe でシングルトン

# 1. TSyringe で DI Container

はじめにでも書いたけど、下記 2 つのサイトを参考にして実際に TSyringe を動かしてみた。
[TypeScript の DI と Tsyringe について](https://zenn.dev/chida/articles/1f7df8f2beb6b6)
[Using tsyringe for dependency injection without using the class syntax - DEV Community 👩‍💻👨‍💻](https://dev.to/leehambley/using-tsyringe-for-dependency-injection-without-using-the-class-syntax-29h7)

コードは、[optimisuke/hello-dicontainer-typescript](https://github.com/optimisuke/hello-dicontainer-typescript)に置いてる。

まず、トップは、こんな感じ。
DI Container を使わかったら、`new User(new Database)`とすべき部分が、`container.resolve(User)`になってる。
中で使ってる、`Database`クラスは、`di.ts`で register してる。

```ts:index.ts
import 'reflect-metadata'
import { container } from './di'
import User from './user'

export const user = container.resolve(User)

user.userId = 1
user.userName = 'yamada'
user.saveUser()
```

中で使うクラスの登録は、`di.ts`に移した。登録した container を再度 export してる。

```ts:di.ts
import { container } from 'tsyringe';
import Database from './database'

export { container };

container.register('IDatabase', {
    useClass: Database
})
```

あと、中身の User と Database はこんな感じ。
デコレーターで指定することで、inject したりされたりできるようにする。

```ts:user.ts
import { injectable, inject } from 'tsyringe'

export interface IDatabase {
    saveUser: (user: User) => void
}

@injectable()
export default class User {
    userId: number = 0
    userName: string = ''

    constructor(
        @inject('IDatabase')
        private database: IDatabase
    ) { }

    saveUser() {
        if (this.userId) {
            this.database.saveUser(this)
        }
    }
}
```

```ts:database.ts
import User, { IDatabase } from "./user";

export default class Database implements IDatabase {
  saveUser(user: User) {
    console.log(`Saved ${user.userName}!`); // Saved yamada!
  }
}
```

設定はこんな感じ。

```json:package.json
{
  "name": "hello-dicontainer-typescript",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "npx ts-node ./index.ts",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "reflect-metadata": "^0.1.13",
    "tsyringe": "^4.7.0"
  }
}
```

```json:tsconfig.json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

以上、ほとんどコピペで恐縮やけど、register 部分を分割してる。
依存関係は事前に register で登録しておいて、使うときに resolve でインスタンスを作ってもらう感じで実装していくイメージ。

# 2. TSyringe でシングルトン

はじめにでも書いたが、以下の記事がすごくわかりやすかったので、TSyringe で試してみた。
[Node で書く時 TypeScript で DI したい | CYOKODOG](https://www.cyokodog.net/blog/inversify-js/)

register する部分が雰囲気違うけど、デコレーター使う部分とかはほぼほぼ一緒。

コードは[optimisuke/hello-ts-tsyringe-singleton](https://github.com/optimisuke/hello-ts-tsyringe-singleton)に置いてる。

まず、`Katana`クラスと`Samurai`クラスはこんな感じ。
シンプルにするためか、インターフェースは使ってない。
（Samurai が Katana に依存しちゃってる（依存性逆転の原則ができてない）けど、Samurai は Katana を大事にしてるやろし、それはそれでいいんじゃないかという気もする。知らんけど。）

```ts:service.ts
import { injectable, inject, singleton } from "tsyringe";

@injectable()
export class Katana {
    sound = "ブンッ！";

    swing() {
        return this.sound;
    }
}

// @injectable()
@singleton()
export class Samurai {
    constructor(
        @inject('Katana')
        private katana: Katana
    ) { }

    fight() {
        return this.katana.swing();
    }

    pwoerUp() {
        this.katana.sound = "ブウウウウンッ！！";
    }
}
```

メインの`index.ts`では`Katana`を register して、`samurai`を resolve してる。
resolve で、2 回、`Samurai`インスタンスを取得してるけど、`@singleton()`をつけてるので、同じものになる。
なので、片方を`powerUp()`するともう片方も`powerUp()`してて期待通り。

```ts:index.ts
import "reflect-metadata";
import { container, injectable } from "tsyringe";
import { Katana } from "./service";
import { Samurai } from "./service";


container.register('Katana', {
    useClass: Katana
})

const samuraiA = container.resolve(Samurai)
const samuraiB = container.resolve(Samurai)

console.log(samuraiA.fight()); // ブンッ！
samuraiA.pwoerUp();
console.log(samuraiA.fight()); // ブウウウウンッ！！
console.log(samuraiB.fight()); // ブウウウウンッ！！ (シングルトンのとき)
```

ついでに、設定はこんな感じ。

```json:package.json
{
  "name": "hello-ts-tsyringe-singleton",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "npx ts-node ./index.ts",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "reflect-metadata": "^0.1.13",
    "tsyringe": "^4.7.0"
  }
}
```

# おわりに

DI Container 食わず嫌いな感じだったけど、自分で動かしながら動作見てたら、だいぶ理解が進んだ。
そのうち、もうちょい、本格的に使ってみたい。
