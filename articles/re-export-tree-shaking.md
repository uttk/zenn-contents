---
title: 'Tree Shaking 的に re-export はやめたほうがいいかもね'
emoji: '🦆'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['typescript', 'treeshaking']
published: true
---

:::message
この記事では TypeScript かつフロントエンドで動作する環境を想定しています。
そのため、この記事の内容は主にフロントエンドからの視点として見て頂ければ幸いです。
:::

# 結論

Tree Shaking はグローバルな処理( ファイル内では未使用でない )に対して有効に働かない場合があります。ファイルサイズを小さくしたいのであれば、`re-export` などを使用せずに [信頼できる唯一の情報源](https://ja.wikipedia.org/wiki/信頼できる唯一の情報源) に従った方が良いと思います。

`re-export` してしまうと、他のファイルにグローバルな処理が書かれていた場合、import 先で使用されていなくてもバンドルされる可能性がありますが、どうしても re-export したい場合は、export 先のファイルにグローバルな処理を書かないようにしたほうが良いと思います 👇

```ts:bad.ts
import { anythingFunction } from "example-library"

anythingFunction(); // 🙅‍♂️ re-export した時にバンドルに含まれる可能性があります！

export const sayHoge = () => console.log("Hoge");
```

```ts:good.ts
import { anythingFunction } from "example-library"

export const sayHoge = () => {

  anythingFunction(); // 🙆‍♂️ sayHoge が使用されないかぎり、バンドルに含まれません！

  console.log("Hoge");
}
```

:::message
ライブラリ側にグローバルな処理があった場合は、どんなに頑張っても re-export をすると不要なコードが入ってしまうので、やるにしてもライブラリの依存度が高くない場合に使用したほうが良さそうです。
:::

# re-export とは？

ここでいう `re-export` とは、export されている値を再度別のファイルで export することを言います。例としては以下のようになります 👇

```ts:./hoge.ts
export const hoge = "Hoge"
```

```ts:./index.ts
// hoge.ts 内の全ての値を export する場合
export * from "./hoge";

// 一部の値だけを export する場合
export { hoge } from "./hoge";
```

この `re-export` ですが、ライブラリなどでは良く使用されていて、使ったことが無くても意外と身近で使われていたります。

とくに有名な書き方として Collect パターン( 私が勝手に名付けた 😇 )というものがあります 👇

```:サンプルコードのディレクトリ構造
.
├── utils
│   ├── a.ts
│   ├── b.ts
│   └── index.ts
└── hoge.ts
```

```ts:./utils/index.ts
export { a } from "./a";
export { b } from "./b";
```

```ts:./hoge.ts
import { a } from "./utils";

a(); // utils で import できる！
```

上記のように、`./utils` 内のソースコードを `index.ts` のようなファイルにまとめて `re-export` することによって、import する側では一つのパスで、そのディレクトリ内のファイルにアクセスすることができます。

この Collect パターンは個別にファイルを import するよりも変更に強いため、非常に便利な構成となっています。

# なぜ Tree Shaking 的によくないのか？

`re-export` は、複数のファイルを一つにまとめている関係上、記述したファイル全てに参照している(import している)状態となります 👇

```ts:前項のCollectパターンと同じディレクトリ構造の場合
import { a } from './utils'

// 👇 以下と同義

import { a } from './utils/a'
import {} from './utils/b'
```

そのため、仮に `./utils/b.ts` にグローバルな処理が書かれている場合は、import した時点でグローバルな処理は読み込まれるため、export している値が無くてもグローバルな処理自体は含まれてしまいます 👇

```ts:./utils/b.ts
// このファイルを import するだけでビルドに含まれる
console.log("hoge");

// 明示的に import しないとビルドに含まれない
export const b = () => console.log("b");

// 未使用の変数などの場合は、そもそもビルドに含まれない
const text = "Hello World!"
```

上記の例では、修正可能な範囲でグローバルな処理が書かれているので、注意していればこの問題は防げるかもしれません。

しかし、**グローバルな処理がライブラリ側に掛かれていた場合は、`re-export` を使っている限りこの問題を回避できません！**

なので、 Tree Shaking 的というか筆者の経験からは、 `re-export` はあんまり使わない方が良いかもと思っています。

# 参考

この記事を書くにあたって、以下の記事を参考にさせて頂きました！
この場を借りてお礼申し上げます。ありがとうございました！

https://zenn.dev/mizchi/scraps/fee64e76afc10d

https://qiita.com/soarflat/items/755bbbcd6eb81bd128c4

# あとがき

この記事では `re-export` を使わない方が良いと言っていますが、ビルドサイズを気にしないのであれば気にせず使っても良いと思います。実際、Collect パターンは便利なので...

ただ私の環境では、致命的な問題だったので、この場を借りて共有した次第です。

これが誰かの参考になれば幸いです。
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。

それではまた 👋
