---
title: 'Next.js で Props をソートすると gzip 時のビルドサイズを少しだけ減らせる'
emoji: '🤏'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['next', 'react', 'eslint']
published: false
publication_name: team_zenn
---

## どういうこと？

少し前に、CSS プロパティを自動ソートするとビルドサイズ(gzip)を減らせる記事を見ました 👇

https://zenn.dev/ubie_dev/articles/c2cdbf76f4d432

これにならい、JSX の Props もソートしたら同じようになるんじゃね？って思って試したら、ビルドサイズを減らすことができたので、この場を借りてその知見を共有したいと思います 💪

**検証環境**

| パッケージ名 | バージョン |
| ------------ | ---------- |
| next         | 13.2.4     |
| react        | 18.2.0     |
| react-dom    | 18.2.0     |
| typescript   | 4.9.5      |

## なぜビルドサイズが減るのか？

例えば以下のようなコードがあったとします 👇

```tsx:page/index.tsx
export default function IndexPage() {
  return (
    <>
      <div data-one data-two data-three />
      <div data-three data-two data-one />
    </>
  )
}
```

上記のコードをビルドすると、以下のようなビルド結果になります 👇

```js:上記のコードに対応する部分をビルドファイルから抜粋
function u() {
  return (0, a.jsxs)(a.Fragment, {
    children: [
      (0, a.jsx)("div", {
        "data-one": !0,
        "data-two": !0,
        "data-three": !0,
      }),
      (0, a.jsx)("div", {
        "data-three": !0,
        "data-two": !0,
        "data-one": !0,
      }),
    ],
  });
}
```

ビルド結果を見ると、ビルド元のコードと**同じ順番で** Props が定義されています。
これはつまり、コード上の Props 順番はビルド結果に影響することになります。

そのため、コード上で Props 順番を揃えておくことで gzip の圧縮率を高めることができ、結果的にビルドサイズを減らすことができます。

実際に gzip 時のファイルサイズを計測すると、ファイルサイズが小さくなっていることが確認できました 👇

|              | ファイルサイズ | gzip 後のファイルサイズ |
| ------------ | -------------: | ----------------------: |
| 並び替える前 |           515B |                   299 B |
| 並び替えた後 |           515B |                   292 B |

## Props を自動ソートするには？

さて、Props をソートするだけでビルドサイズが減らせられるとなれば、皆さんもソートしたくてウズウズしますよね？

ありがたいことに、eslint-plugin-react の [jsx-sort-props](https://github.com/jsx-eslint/eslint-plugin-react/blob/master/docs/rules/jsx-sort-props.md) を有効にする事で自動ソートが可能になります！🙌 ｲｴｰｲ

`eslint-config-next` を使っている場合は既に eslint-plugin-react が設定されているので、[jsx-sort-props](https://github.com/jsx-eslint/eslint-plugin-react/blob/master/docs/rules/jsx-sort-props.md) を有効にするだけで自動ソートすることができます 👇

```diff js:.eslintrc.json
 {
   "extends": "next/core-web-vitals",
   "rules": {
+    "react/jsx-sort-props": "warn"
   }
 }
```

上記の設定で eslint を実行して、アルファベット順に Props がソートされていればオッケーです 👌

## あとがき

ここまで読んでくれてありがとうございます。

冒頭で紹介した記事に着想を得てこの記事が書けたので、著者である [@jimbo](https://zenn.dev/jimbo) さんには感謝ですね 🙏 ｱﾘｶﾞﾀﾔｰ

Next.js 以外でも Props の順番がビルド結果に影響している場合は同じようにビルドサイズが減らせられると思うので、Props の順番にこだわりなど無い方は一度試してみてはいかがでしょうか？( おそらく、Next.js よりも SPA の方が良い結果が出やすいと思います )

ちなみに Zenn の環境では、なぜか 10 バイト増えてしまったので、なんか敗北した気持ちになってます 😇 ﾄﾞｳｼﾃ

記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。
これが誰かの参考になれば幸いです。

それではまた 👋
