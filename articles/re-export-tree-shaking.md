---
title: 'Tree Shaking に対応してないライブラリを使っている場合は re-export はやめたほうがいいかも'
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

[Tree Shaking](https://developer.mozilla.org/ja/docs/Glossary/Tree_shaking) に対応してないライブラリを使っているソースコードで `re-export` を使用していると、不用意にファイルサイズを大きくしてしまう可能性があります。そのため、対応してないライブラリを使用している状況下である場合 ( またはそのような状況になりそうな場合 ) には、`re-export` などを使用せずに [信頼できる唯一の情報源](https://ja.wikipedia.org/wiki/信頼できる唯一の情報源) に従った方が良いと思います。

どうしても `re-export` 使用したい場合、副作用があるファイルが分かっているのであれば、`sideEffects` オプションを使用することで影響を最小限に抑えることができます 👇

```json:package.json
{
  /* -- 省略 -- */

  // 全ての副作用を無効にする
  "sideEffects": false

  /* -- 省略 -- */
}
```

```json:package.json
{
  /* -- 省略 -- */

  // 特定のファイル、ライブラリの副作用を指定します
  // これにより、指定されたファイルの副作用は含まれるようになります
  "sideEffects": [
    "./src/side-effects.ts",
    "./src/side-effects/**",
  ],

  /* -- 省略 -- */
}
```

:::message
上記の設定は、副作用があるライブラリ内部のソースコードには有効になりませんので、ご注意ください！副作用があるライブラリを使用しているファイルなどに対して設定するようにしましょう！
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

# なぜ re-export はやめたほうがいいのか？

冒頭でも記述しましたが、[Tree Shaking](https://developer.mozilla.org/ja/docs/Glossary/Tree_shaking) に対応してないライブラリを使っているソースコードで、`re-export` を使用していると、意図せず使っていないソースコードをビルド結果に含めてしまう可能性があります。

これは、 `re-export` が複数のファイルを一つにまとめている関係上、記述したファイル全てに参照している( import している )状態となるためです 👇

```ts:前項のCollectパターンと同じディレクトリ構造の場合
import { a } from './utils'

// 👇 以下と同義

import { a } from './utils/a'
import {} from './utils/b'
```

そのため、仮に `./utils/b.ts` で副作用があるライブラリを読み込んでいる場合は、import した時点で副作用が読み込まれてしまうため、import している値が無くても副作用自体は含まれてしまいます 👇

```ts:./utils/b.ts
// グローバルな処理(副作用)があるライブラリ
// b.ts を import したファイルは、b.ts のソースコードを使用してなくても、
// import した時点で "side-effects-library" の副作用を含めてしまいます
import { printText } from "side-effects-library"

// 明示的に import しないとビルドに含まれません
export const b = () => printText("b");
```

# 対処法

対処法として package.json に `sideEffects` というフィールドを設定することで、副作用があるファイルを webpack などのバンドラーに知らせることができます 👇

```json:package.json
{
  /* -- 省略 -- */

  // 副作用がないということをバンドラーに知らせる
  "sideEffects": false,

  /* -- 省略 -- */
}
```

```json:package.json
{
  /* -- 省略 -- */

  // 副作用があるファイルが分かる場合は指定する事もできます。
  // これによって、ここで指定していないファイルの副作用は無視されます。
  "sideEffects": [
    "./src/side-effects.ts",
    "./src/side-effects/**",
  ],

  /* -- 省略 -- */
}
```

上記の設定によって副作用があるファイルを設定できるので、自分たちが把握できる副作用に関しては Tree Shaking を有効にすることができます。

しかし、どのライブラリに副作用があるかを把握するのは至難の業であるため、対処ができると言っても限界があります。

なので、筆者個人の考えとしては `re-export` をむやみやたらに使うのではなく、副作用が把握できる範囲で使用したほうがいいと考えています。

# 参考

この記事を書くにあたって、以下の記事を参考にさせて頂きました！
この場を借りてお礼申し上げます。ありがとうございました！

https://zenn.dev/mizchi/scraps/fee64e76afc10d

https://qiita.com/soarflat/items/755bbbcd6eb81bd128c4

https://www.kabuku.co.jp/developers/tree-shaking-in-2018

https://webpack.js.org/guides/tree-shaking/#mark-the-file-as-side-effect-free

https://dwango-js.github.io/performance-handbook/startup/module-field/

# あとがき

この記事では `re-export` を使わない方が良いと言っていますが、ビルドサイズを気にしないのであれば気にせず使っても良いと思います。実際、Collect パターンは便利ですし、Tree Shaking に対応したライブラリのみ使用している場合は、特に気にする必要のない話題です。ただ私の環境では対応してないライブラリを使用しており、致命的な問題だったので、この場を借りて共有した次第です。

また、記事の内容に対してご指摘頂いた [@smikitky](https://zenn.dev/smikitky) さんには、この場を借りて感謝申し上げます。ありがとうございました 🙇‍♂️🙇‍♀️

これが誰かの参考になれば幸いです。
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。

それではまた 👋
