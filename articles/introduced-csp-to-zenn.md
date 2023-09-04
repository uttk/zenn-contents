---
title: "Zenn に段階的に Content Security Policy を導入した話"
emoji: "🍤"
type: "tech"
topics: ["security", "csp", "nextjs"]
published: false
publication_name: team_zenn
---

## この記事について

先日、Zenn では [Content Security Policy](https://developer.mozilla.org/ja/docs/Web/HTTP/CSP) を導入しました。

https://info.zenn.dev/2023-07-18-csp

この記事では [Content Security Policy](https://developer.mozilla.org/ja/docs/Web/HTTP/CSP) を [Next.js ( Pages Router )](https://nextjs.org/) で導入する方法を解説するともに、Zenn の実例を紹介したいと思います。

## Content Security Policy とは？

そもそも Content Security Policy を知らない人が居るかもしれません。

Content Security Policy ( 以後 CSP と表記 )とは、ブラウザに備わっている機能の一つで、この機能を使うことで設定したサイト内のセキュリティリスクを軽減することができます。

https://developer.mozilla.org/ja/docs/Web/HTTP/CSP

基本的には導入した方がいいのですが、設定項目が多いうえに少し設定を間違えるとサイトが機能しなくなったりするので、導入コストがけっこう高いです。そのため、導入できているサイトは意外と少ない印象です。[^1]

[^1]: https://scotthelme.co.uk/top-1-million-analysis-june-2022/ の調査内容を参照

ただ、導入するメリットはすごくあるので、もし CSP を導入してみたい方は上記の MDN のドキュメントを読みつつ、以下のサイトも読んでおくといいと思います 👇

https://inside.pixiv.blog/kobo/5137

https://techblog.yahoo.co.jp/entry/2023071830429434/

### Content Security Policy Level について

また、CSP は「 Level 」という名前でバージョン管理がされています。この「 Level 」は現在( 2023/09 )までに Level １～３ まで出ており、その内の「 Level 3 」は草案状態となっています。

https://www.w3.org/TR/CSP3/

Level １ ～ ３ の要点は以下のようになります 👇

- **Level 1**
  - CSP の基本的な設定オプション(ディレクティブ)が導入されました
  - 古いバージョンなので現在ではほとんど使われていません(たぶん)
  - 参照: https://www.w3.org/TR/2012/CR-CSP-20121115/
- **Level 2**
  - Level 1 から破壊的な変更と多くの設定オプション(ディレクティブ)が加えられました
  - 2023/09 現在では Level 3 が使えない時のフォールバックとして使うことが多いです
  - 参照: https://www.w3.org/TR/CSP2/
- **Level 3**
  - Level 2 から互換性のある変更が加えられました
    - そのため、部分的な導入が可能です
  - 2023/09 現在は草案状態ですが、多くのモダンブラウザは対応しています
    - そのため、できるなら Level 3 を軸に導入した方が良いです
  - 参照: https://www.w3.org/TR/CSP3/

最新のモダンブラウザでは Level 3 まで対応しています[^2]が、古いブラウザでは Level 2 までしか対応していないことに注意してください。特に Safari は、後述する [Strict CSP](https://csp.withgoogle.com/docs/strict-csp.html) に必要な `strict-dynamic` に 2022/3 まで対応していなかった[^3]ため、対応するならフォールバックなどが必要になります。( Safari のフォールバックについては後述します )

[^2]: 細かな対応状況はブラウザごとに違いがあるので、ちゃんと対応しているとは言い切れませんが...
[^3]: [can i use `strict-dynamic`](https://caniuse.com/?search=strict-dynamic)を参照

以上で CSP の紹介は終わりです。
次は実際に CSP を設定する方法を解説してきます 👉

## Next.js で導入するには

:::message
Zenn では Next.js ( Pages Router ) を使用しているため、この記事では Next.js を使った設定方法を解説します。
:::

あとで Zenn の実例も紹介しますが、その前に CSP の設定方法を簡単に解説しておきます。Next.js で CSP を設定するには基本的に [Custom Document](https://nextjs.org/docs/pages/building-your-application/routing/custom-document) を使って設定します 👇

```tsx:./pages/_document.tsx
import Document, {
  Html,
  Head,
  Main,
  NextScript,
  DocumentContext,
  DocumentInitialProps,
} from 'next/document';
import { createHash } from 'crypto';
import { v4 as uuidv4 } from 'uuid';

interface MyDocumentProps extends DocumentInitialProps {
  nonce?: string;
}

class MyDocument extends Document<MyDocumentProps> {
  /**
   * この`getInitialProps`はクライアントでは実行されず、常にバックエンドのみで実行されます
   */
  static async getInitialProps(ctx: DocumentContext) {
    const initialProps = await Document.getInitialProps(ctx);

    // nonce の値はリクエストごとに変更する
    const nonce = createHash('sha256').update(uuidv4()).disest('base64')

    // ここで CSP の設定をする。
    // SSG しているページではヘッダーが設定できないことに注意！
    ctx.res.setHeader('Content-Security-Policy', /*...*/);

    return { ...initialProps, nonce };
  }

  render() {
    const nonce = this.props.nonce;
    return (
      <Html lang="ja">
        <Head nonce={nonce} />
        <body>
          <Main />
          <NextScript nonce={nonce} />
        </body>
      </Html>
    )
  }
}

export default MyDocument;
```

上記のように `getInitialProps` を使って CSP を設定することで、 SSG 以外のページで反映することができます。( コメントにも書きましたが、[Custom Document](https://nextjs.org/docs/pages/building-your-application/routing/custom-document) の `getInitialProps` はクライアントでは実行されず、常にバックエンドのみで実行されます[^4] )

[^4]: [Next.js の公式ドキュメント](https://nextjs.org/docs/pages/building-your-application/routing/custom-document#:~:text=getInitialProps%20in%20_document%20is%20not%20called%20during%20client%2Dside%20transitions.)を参照

また、CSP はレスポンスヘッダーに設定する方法と `<meta />` を使って設定する二通りの方法がありますが、`<meta />` を使うと [report-to](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Security-Policy/report-to) などの段階的に導入するための機能が使えないため、Zenn ではレスポンスヘッダーに設定する方法を取っています 👇

```tsx:CSPをレスポンスヘッダーに設定
// ここで CSP の設定をする。
// SSG しているページではヘッダーが設定できないことに注意！
ctx.res.setHeader('Content-Security-Policy', /*...*/);
```

SSG しているページでは CSP が設定できないという問題点はありますが、SSG しているページでは動的なコンテンツが無くセキュリティリスクが低いので、現時点では許容範囲としています。

あと、Next.js は nonce 方式に対応しているため `getInitialProps` で nonce を生成して、`<Head />` と `<NextScript />` に Props として渡してあげるだけで Next.js が生成する要素に nonce を設定できます 👇

```tsx:リクエストごとにnonceを生成する
  static async getInitialProps(ctx: DocumentContext) {
    const initialProps = await Document.getInitialProps(ctx);

    // nonce の値はリクエストごとに変更する
    const nonce = createHash('sha256').update(uuidv4()).disest('base64')

    // 生成した nonce をヘッダーに設定する
    ctx.res.setHeader('Content-Security-Policy', `'nonce-${nonce}'`);

    // render()で使うので props に含める
    return { ...initialProps, nonce };
  }
```

```tsx:生成したnonceをpropsで受け取って設定する
  render() {
    const nonce = this.props.nonce; // getInitialProps() で生成したnonce文字列
    return (
      <Html lang="ja">
        <Head nonce={nonce} />
        <body>
          <Main />
          <NextScript nonce={nonce} />
        </body>
      </Html>
    )
  }
```

上記のポイントとして、リプレイ攻撃などを防ぐために nonce は固定値ではなくリクエストごとに生成してヘッダーに設定することが推奨されています。なので、リクエストごとに実行される `getInitialProps()` で毎回 nonce を生成するようにしています。

はい、Next.js の基本的な設定方法の解説はこれで終わりです。
ここから先はサイトにより設定値が変わってくるため、次項では Zenn がどのように設定しているかの実例を解説していきます 🌏

## Zenn ではどのように導入したか

Zenn では Google Strict が提唱する [Strict CSP アプローチ](https://csp.withgoogle.com/docs/strict-csp.html) になるべく対応するため、基本的には `nonce` + [`strict-dynamic`](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Security-Policy/script-src#strict-dynamic) で対応するようにし、それが無理な場合はホワイトリスト方式で対応しています。

また、設定するディレクティブは基本的に `script-src` のみ設定しています。本来であれば `style-src` や `font-src` を指定するべきですが、設定しても効果が薄く、またユーザーへの影響範囲が大きいことから設定していません。

はい、前置きはこのくらいにして、まずは nonce の基本的な設定を見てきましょう 👉

### nonce を設定する

nonce に設定する文字列を生成し、Next.js が生成する `<script />` や `<style />` に nonce 値を設定するには以下のようにします 👇

```tsx:./pages/_document.tsx
import Document, {
  Html,
  Head,
  Main,
  NextScript,
  DocumentContext,
  DocumentInitialProps,
} from 'next/document';
import clsx from 'clsx';
import { createHash } from 'crypto';
import { v4 as uuidv4 } from 'uuid';

const isDev = process.env.NODE_ENV === 'development';

interface MyDocumentProps extends DocumentInitialProps {
  nonce?: string;
}

class MyDocument extends Document<MyDocumentProps> {
  static async getInitialProps(ctx: DocumentContext) {
    const initialProps = await Document.getInitialProps(ctx);

    // <script /> や <style /> に設定する nonce 文字列を作成
    const nonce = createHash('sha256').update(uuidv4()).digest('base64');

    // ここで CSP の設定をする。
    // SSG しているページではヘッダーを設定できないことに注意！
    ctx.res.setHeader(
      'Content-Security-Policy',
      [
        clsx(
          'script-src',
          {
            // 開発環境では eval などを使用しているため無効にする
            "'unsafe-eval'": isDev,
            "'unsafe-inline'": isDev,
          },
          [
            "'self", // 自分自身のスクリプトを許可する
            "'https:'", // Safari のフォールバック用
            "'strict-dynamic'", // 許可したスクリプトが生成したスクリプトを許可する
            `'nonce-${nonce}'`, // nonce が設定されたスクリプトを許可する
          ],
        )
      ].join(";") + ";",
    );

    // render() で nonce を使うので Props に含める
    return { ...initialProps, nonce };
  }

  render() {
    const nonce = this.props.nonce;
    return (
      <Html lang="ja">
        <Head nonce={nonce} />
        <body>
          <Main />
          <NextScript nonce={nonce} />
        </body>
      </Html>
    )
  }
}

export default MyDocument;
```

本来の使用用途とは違いますが、CSP を設定する時に [clsx](https://www.npmjs.com/package/clsx) などの文字列を連結するパッケージを使用すると制御しやすいのでオススメです。( ~~でも、本来の使用用途と違い過ぎるので、やっぱり使わない方がいい気がしてｋ~~)

また、Next.js は開発環境ではホットリロードなどの関係で eval やインラインスクリプトを多用するため、開発環境では `'unsafe-eval'` と `'unsafe-inline'` を無効にする必要があります。( じゃないとまともに動きません... )

```tsx:開発環境用に設定する必要があります
  {
    // 開発環境では eval などを使用しているため無効にする
    "'unsafe-eval'": isDev,
    "'unsafe-inline'": isDev,
  },
```

ただし、`'unsafe-inline'` は nonce が有効な場合は無視されるので常に有効にしておいても基本的には問題ないです。`'unsafe-eval'` もできれば無効にしたいですが、無効に出来ない場合は有効にしておきましょう。

そして、その下には nonce を有効にするための設定などを実装しています 👇

```ts:nonceを有効にするための設定
[
  "'self", // 自分自身のスクリプトを許可する
  "'https:'", // Safari のフォールバック用
  "'strict-dynamic'", // 許可したスクリプトが生成したスクリプトを許可する
  `'nonce-${nonce}'`, // nonce が設定されたスクリプトを許可する
]
```

ここは、後でホワイトリスト形式で許可したい URL も含めるようにしたいので配列で定義しています。また、少し古めの Safari のために `https:` フォールバックを設定しています。対応しない場合は削除しても問題ないです。

最後に、生成した nonce 値を `<script />` や `<style />` に設定するために `render()` 内で Props を設定します 👇

```tsx
  render() {
    const nonce = this.props.nonce;
    return (
      <Html lang="ja">
        <Head nonce={nonce} />
        <body>
          <Main />
          <NextScript nonce={nonce} />
        </body>
      </Html>
    )
  }
```

Next.js が CSP の nonce 方式に対応しているため、コンポーネントの Props に nonce 値を渡すだけで実装できます。簡単ですね。

### ホワイトリスト形式で許可する

次はホワイト形式でスクリプトを許可します。本来は使用しない方が良い[^5]ですが、Zenn では [Katex](https://katex.org/) に対応するために使用しています。

[^5]: [Google の調査](https://research.google/pubs/pub45542/)により CSP Level 2 まで推奨とされてきた XSS 対策に脆弱性が指摘されました。

設定するには ホワイトリスト形式で許可したい URL をディレクティブを指定している配列に追加します 👇

```diff tsx:KatexのURLを配列に追加
 [
   "'self", // 自分自身のスクリプトを許可する
   "'https:'", // Safari のフォールバック用
   "'strict-dynamic'", // 許可したスクリプトが生成したスクリプトを許可する
   `'nonce-${nonce}'`, // nonce が設定されたスクリプトを許可する
+  'https://cdn.jsdelivr.net/npm/katex/dist/katex.min.js',
 ]
```

これで完了です。もし他に許可したい URL がある場合は同じように追加すれば対応できます。簡単ですね。

### インラインスクリプトを許可する

次はインラインスクリプトを許可できるようにします。インラインスクリプトを許可するにはソースコードからハッシュ文字列を生成し、そのハッシュ文字列をヘッダーに設定します 👇

```tsx:インラインスクリプトからハッシュを生成してヘッダーに設定する
// ...

type ScriptProps = React.ComponentProps<"script"> & {
  key: string
};

// <script />に渡す Props をまとめた配列
const inlineScriptProps: [] = [
  {
    key: "inline script",
    dangerouslySetInnerHTML: {
      __html: `console.log("これはインラインスクリプトです")`,
    },
  },
];

// インラインスクリプトを安全にするためのハッシュ文字列
const inlineHashList: string[] = inlineScriptProps.flatMap((props) => {
  const code = props.dangerouslySetInnerHTML?.__html as string;
  return code
    ? [`'sha256-${createHash("sha256").update(code).digest("base64")}`]
    : [];
});

// ...

class MyDocument extends Document<NyDocumentProps> {
  static async getInitialProps(ctx: DocumentContext) {
    // ...
    ctx.res.setHeader(
      clsx(
        'script-src',
        // ...
        [
          // ...
          ...inlineHashList, // インラインスクリプトを許可するハッシュ文字列
        ]
      ),
    )
  }

  render() {
    const nonce = this.props.nonce;
    return (
      <Html lang="ja">
        <Head nonce={nonce}>
          {/* 許可したインラインスクリプト */}
          {inlineScriptProps.map((props) => (
            <script {...props} key={props.key} nonce={nonce} />
          ))}
        </Head>
        <body>
          <Main />
          <NextScript nonce={nonce} />
        </body>
      </Html>
    )
  }
}
```

特に難しい所は無いと思いますが、注意点として JSX で Props を指定する時に nonce も設定することを忘れないでください。じゃないとエラーがでます 🥺

```tsx:nonceを設定することを忘れない！
  {inlineScriptProps.map((props) => (
    <script {...props} key={props.key} nonce={nonce} />
  ))}
```

はい、これでインラインスクリプトの設定は完了です。
次は `script-src` 以外のディレクティブを設定していきます 👉

### その他のディレクティブを設定する

Zenn では、`script-src` 以外のディレクティブとしては `object-src` と `base-uri` のみ設定していますので、その設定を `script-src` の下に追加します 👇

```diff tsx
   static async getInitialProps(ctx: DocumentContext) {
     // ...
     ctx.res.setHeader(
       clsx(
         'script-src',
         // ...
       ),
+      // object-src はフラッシュ系のXSS対策に必須
+      `object-src 'none'`,
+      // baseタグの注入によるXSSへの対策に必須
+      `base-uri 'none'`,
     )
   }
```

### 最終的な設定

ここまでの実装をまとめると以下のようになります 👇

```tsx:./pages/_document.tsx
import Document, {
  Html,
  Head,
  Main,
  NextScript,
  DocumentContext,
  DocumentInitialProps,
} from 'next/document';
import clsx from 'clsx';
import { createHash } from 'crypto';
import { v4 as uuidv4 } from 'uuid';

const isDev = process.env.NODE_ENV === 'development';

type ScriptProps = React.ComponentProps<"script"> & {
  key: string
};

// <script />に渡す Props をまとめた配列
const inlineScriptProps: [] = [
  {
    key: "inline script",
    dangerouslySetInnerHTML: {
      __html: `console.log("これはインラインスクリプトです")`,
    },
  },
];

// インラインスクリプトを安全にするためのハッシュ文字列
const inlineHashList: string[] = inlineScriptProps.flatMap((props) => {
  const code = props.dangerouslySetInnerHTML?.__html as string;
  return code
    ? [`'sha256-${createHash("sha256").update(code).digest("base64")}`]
    : [];
});

interface MyDocumentProps extends DocumentInitialProps {
  nonce?: string;
}

class MyDocument extends Document<MyDocumentProps> {
  static async getInitialProps(ctx: DocumentContext) {
    const initialProps = await Document.getInitialProps(ctx);

    // <script /> や <style /> に設定する nonce 文字列を作成
    const nonce = createHash('sha256').update(uuidv4()).digest('base64');

    // ここで CSP の設定をする。
    // SSG しているページではヘッダーを設定できないことに注意！
    ctx.res.setHeader(
      'Content-Security-Policy',
      [
        clsx(
          'script-src',
          {
            // 開発環境では eval などを使用しているため無効にする
            "'unsafe-eval'": isDev,
            "'unsafe-inline'": isDev,
          },
          [
            "'self", // 自分自身のスクリプトを許可する
            "'https:'", // Safari のフォールバック用
            "'strict-dynamic'", // 許可したスクリプトが生成したスクリプトを許可する
            `'nonce-${nonce}'`, // nonce が設定されたスクリプトを許可する
            'https://cdn.jsdelivr.net/npm/katex/dist/katex.min.js',
            ...inlineHashList, // インラインスクリプトを許可するハッシュ文字列
          ],
        ),

        // object-src はフラッシュ系のXSS対策に必須
        `object-src 'none'`,

        // baseタグの注入によるXSSへの対策に必須
        `base-uri 'none'`,
      ].join(";") + ";",
    );

    // render() で nonce を使うので Props に含める
    return { ...initialProps, nonce };
  }

  render() {
    const nonce = this.props.nonce;
    return (
      <Html lang="ja">
        <Head nonce={nonce}>
          {/* 許可したインラインスクリプト */}
          {inlineScriptProps.map((props) => (
            <script {...props} key={props.key} nonce={nonce} />
          ))}
        </Head>
        <body>
          <Main />
          <NextScript nonce={nonce} />
        </body>
      </Html>
    )
  }
}

export default MyDocument;
```

## 段階的に導入するには？

ここまで CSP の導入方法について解説してきましたが、これから CSP を導入する場合、上記の設定をいきなり導入するのは流石に難しいと思います。

ですが、CSP には `Report-Only` というサイトの動作は変えずに設定したセキュリティ要件に違反した時に指定した URL に違反内容を送信してくれる機能があります 👇

https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy-Report-Only

この機能を使うことで、

1. `Content-Security-Policy-Report-Only` を設定
2. Report-Only で報告された違反内容を確認
3. 違反内容に問題があれば修正
4. 違反が報告されなくなったら `Content-Security-Policy` を本番に適用

といった感じで、段階的に導入することが可能です。

また、設定も簡単で以下のように実装できます 👇

```tsx:Report-Onlyとして設定する
  static async getInitialProps(ctx: DocumentContext) {
    ctx.res.setHeader(
      'Content-Security-Policy-Report-Only',
      [
        /* -- 省略 -- */
        // 違反内容を受け取るエンドポイントURLを設定
        `report-uri https://csp-report-logger.example.com`,
      ].join(";") + ";",
    );
  }
```

実は上記で設定している [report-uri](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Security-Policy/report-uri) は非推奨なんですが、移行先の [report-to](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Security-Policy/report-to) が Google Chrome をはじめとしたモダンブラウザで上手く動作しないことがありましたので、泣く泣く[report-uri](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Security-Policy/report-uri) を使用しています 😥 ( もちろん、いずれは [report-to](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Security-Policy/report-to) に対応します )

そして、正しく設定できていれば以下のような JSON がエンドポイントに送られます 👇

```json5:report-uriに設定したエンドポイントが受け取る違反内容のJSON
{
  "csp-report": {
    "document-uri": "", // 違反が行われたページのURL
    "referrer": "", // 違反が行われたページのリファラ
    "violated-directive": "", // 違反したディレクティブ(フォールバックが適用された場合は、そちらのディレクティブを表示)
    "effective-directive": "", // 違反したディレクティブ
    "original-policy": "", // 設定しているポリシーの全文
    "disposition": "", // ポリシーの適用方法
    "blocked-uri": "", // 違反したリソースのURL
    "status-code": 0, // グローバルオブジェクトがインスタンス化されたリソースの HTTP ステータスコード( よく分からん )
    "script-sample": "", // 違反したインラインのscriptやstyleの最初の40文字
  }
}
```

コメントだけだと分かりにくいと思いますので、実際の値も紹介しておきます 👇

```json:実際のCSP違反レポートの内容
{
  "csp-report": {
    "document-uri": "https://zenn.dev",
    "referrer": "https://www.google.com/",
    "violated-directive": "script-src-elem",
    "effective-directive": "script-src-elem",
    "original-policy": "script-src 'nonce-xxx';report-uri https://csp-report-logger.example.com;",
    "disposition": "enforce",
    "blocked-uri": "https://xss.example.com",
    "status-code": 200,
    "script-sample": "",
  }
}
```

基本的に見るべき箇所は、

- `referrer`: 違反が行われたページのリファラ
- `blocked-uri`: ブロックされたリソースの URL
- `violated-directive` or `effective-directive`: 違反したディレクティブ
- `document-uri`: 違反が行われたページの URL

になります。ただ、違反の中には対応しなくても良い違反なども含まれているので、なるべく多くの違反を集めて総合的に判断するといいと思います。

### CSP の違反レポートは Cloud Functions で受け取ると便利

Zenn でも `Content-Security-Policy-Report-Only` を使って違反内容を確認しながら段階的に導入しましたが、[report-uri](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Security-Policy/report-uri) に設定するエンドポイントを Cloud Functions で実装し、違反内容をコンソールログに出力することで Cloud Logging で細かく検索できてとても便利でした。

https://cloud.google.com/logging?hl=ja

## 参考

https://developer.mozilla.org/ja/docs/Web/HTTP/CSP

https://inside.pixiv.blog/kobo/5137

https://csp.withgoogle.com/docs/strict-csp.html

https://techblog.yahoo.co.jp/entry/2023071830429434/

https://research.google/pubs/pub45542/

https://kotamat.com/post/nextjs-strict-csp/

https://zenn.dev/monicle/articles/aee4871b288264

## あとがき

ここまで読んでくれてありがとうございます 🙏

CSP の設定自体はそこまで難しくないんですが、やっぱり影響範囲が大きくて、それを考慮しながら導入するのがとても大変でした。この辺りはチームメンバーが色々と手伝ってくれて、とても助かりました。本当にありがとうございました 🙌

また、CSP の導入は [catnose](https://zenn.dev/catnose99) さんが提案してくれたことによって、導入に踏み切れたのも大きかったです。今後 Zenn が成長してサービスが大きくなればなるほど導入が難しくなるので、このタイミングで導入できたのはとても良かったと思います。

記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。
これが誰かの参考になれば幸いです。

それではまた 👋
