---
title: "Next.jsの環境変数(ステージング)は気をつけたほうが、いいよ。"
emoji: "🧸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

# 結論

Next.js では組み込みで環境変数を実行環境ごとに定義することができます[^1] が、正しく設定しないと環境変数が正しく埋め込まれず、Tree Shaking などが有効に働かない可能性があります。

[^1]: https://nextjs.org/docs/basic-features/environment-variables

具体例を挙げると、以下のように一方のファイルのみに環境変数が定義されているとします 👇

```shell:.env.development
NEXT_PUBLIC_HOGE="hoge-development"
NEXT_PUBLIC_IS_DEV=true
```

```shell:.env.production
NEXT_PUBLIC_HOGE="hoge-production"
# 本番では`NEXT_PUBLIC_IS_DEV`を使わないので定義していないとする
```

この状態で、以下のコードを定義します。

```ts:src/index.ts
console.log(process.env.NEXT_PUBLIC_HOGE)

if( proecss.env.NEXT_PUBLIC_IS_DEV ) {
  console.log("The current env is Development")
}
```

上記のコードを `development`・`production` それぞれの環境でビルドすると以下のようになります 👇

```js:developmentの場合
console.log("hoge-development")

if( true ) {
  console.log("The current env is Development")
}
```

```js:productionの場合
console.log("hoge-production")

if( proecss.env.NEXT_PUBLIC_IS_DEV ) { // 本来は`undefined`などの値に置き換えて欲しかった...
  console.log("The current env is Development")
}
```

上記を見て分かる通り、両方の env ファイルに定義している `NEXT_PUBLIC_HOGE` は正しく置き換えが機能していますが、`.env.development` のみで定義されている `NEXT_PUBLIC_IS_DEV` は、prodution 環境では置き換えが発生せず、`process.env.NEXT_PUBLIC_IS_DEV` のままになっています。

これを回避するには、`.env.production` 内に `NEXT_PUBLIC_IS_DEV` を定義してあげることで正しく置き換えられるようになります 👇

```shell:.env.production
NEXT_PUBLIC_HOGE="hoge-production"
NEXT_PUBLIC_IS_DEV=false
```

## 原因

Next.js の実装を見るに、env ファイルに定義されている値のみを [DefinePlugin](https://webpack.js.org/plugins/define-plugin/) に設定しているようです 👇

https://github.com/vercel/next.js/blob/210fa39961033842303495e86bf1658dcca01b24/packages/next/build/webpack-config.ts#L1442-L1451

そのため、**env ファイルの値はなるべくは省略せず、適切に定義する必要がある**ようです。

# あとがき

私は、今の今まで未定義の環境変数は `undefined` に置き換えられると思い込んでいましたが、皆さんはどうだったでしょうか？

気をつけないと、開発環境のコードが本番に含まれてしまって、パフォーマンスやセキュリティに影響が出ているかもしれませんので、もし心当たりがある人は今一度ビルド結果などを確認したほうがいいと思いますよ。( もちろん、私はやらかしました 😇 )

これが誰かの参考になれば幸いです。
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。

それではまた 👋
