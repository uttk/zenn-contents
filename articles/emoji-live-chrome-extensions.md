---
title: "画面キャプチャを仮想カメラとして扱えるようにするChrome拡張を作ってみる"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["chromeextension", "typescript"]
published: true
---

# この記事について

https://twitter.com/catnose99/status/1445612717356638212

先日、[@catnose](https://zenn.dev/catnose99)さんがカメラ映像の代わりに絵文字(Emoji)を配信するためのサービスを公開されました。凄く完成度が高くて良いサービスだと思ったので、さっそく使ってみたのですが、仮想カメラとして使用するためには [OBS Studio](https://obsproject.com/ja) が必要でした。[^1]

[^1]: https://live.catnose99.com/guide を参照

サービスを使うには全然申し分無いのですが、「 もっと簡単にできたらなぁ～ 」と思ってしまうのが私の悪い所で、すぐさまブラウザのみでどうにかできないかと調べてみると、**色々な制約はありますが**、Chrome 拡張を使うことで [OBS Studio](https://obsproject.com/ja) を使わずとも仮想カメラを使用できることが分かりました。

実装も簡単にできるので、 今回は [Google Meet](https://apps.google.com/intl/ja/meet/) で、画面キャプチャを仮想カメラとして表示する Chrome 拡張を作って行こうと思います 💪

:::message
実装は TypeScript を使って実装してきます。
:::

# 今回作るモノについて

https://github.com/uttk/screen-capture-virtual-camera

今回この記事で作る Chrome 拡張は、上記の Github リポジトリで公開しています。
この記事を読むのが面倒な人は、上記のリポジトリからソースコードを直接読むと良いと思います。( ソースコードも少ないので、謙虚に 9 分で読めます )

# Chrome 拡張を作ってみる

それでは、さっそく作って行きましょう 💪
やることとしては、

1. **仮想カメラを使いたいサイト( 今回は Google Meet )で、任意のスクリプトを挿入する**
2. **特定の API をラップして、仮想カメラを扱えるようにする**

この二点だけですので、ソースコードを見せながら解説していこうと思います。まずは、任意のスクリプトを挿入するところから行きましょー 🎳

# 任意のスクリプトを挿入する処理を実装する

Chrome 拡張には、三つの世界線( `content_scripts`・`background`・`page_action` )があり、この内の `content_scripts` では、表示されるサイトにアクセスすることができます。これを~~悪用~~使うことで、任意の処理を対象のサイトで実行する事ができます。

今回は、[Google Meet](https://apps.google.com/intl/ja/meet/) の `<head />` に `<script />` を挿入して、任意のファイル(`./src/main.ts`)を実行する処理を書いて行きます。`./src/index.ts` ファイルを作成して、以下のように実装しましょう 👇

```ts:./src/index.ts
// <script /> を作成
const script = document.createElement("script");

// 実行したいファイルを設定する
script.setAttribute("type", "module");
script.setAttribute("src", chrome.extension.getURL("main.js"));

// サイトの `<head />` を取得する
const head =
  document.head ||
  document.getElementsByTagName("head")[0] ||
  document.documentElement;

// `<script />` を挿入する
head.insertBefore(script, head.lastChild);
```

注意点としては、`<script />` に設定する `src` にはビルド後のファイル名を指定する必要があります。今回の場合は、`./src/main.ts` を `main.js` でビルドしますので、`main.js`の方を設定します。

上記の実装ができましたら、次は画面キャプチャ処理を実装してきます 🎥

# 画面キャプチャを実行する

画面キャプチャを実行するためには、[MediaDevices.getDisplayMedia()](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia) というブラウザ API を呼び出すことで、画面キャプチャの動画情報(MediaStream)を取得することができます。

今回は、その動画情報(MediaStream)を仮想カメラとして扱いたいので、その情報を返す関数を `./src/main.ts` に実装します 👇

```ts:./src/main.ts
/**
 * @description 画面キャプチャのStreamを返す関数
 */
const getCaptureStream = async () => {
  // 画面キャプチャのStreamを取得する
  const captureStream = await navigator.mediaDevices.getDisplayMedia({
    audio: false, // 音声はいらないので false に。
    video: true
  });

  if (!captureStream) throw new Error("動画情報の取得に失敗しました 😥");

  // キャプチャが終了した時のコールバックを設定する
  captureStream.getTracks().forEach((track) => {
    track.onended = () => console.log("STOP: Screen Capture Virtual Camera 🎥");
  });

  return captureStream;
};
```

上記で定義した関数を実行すると、以下のようなポップアップが表示され、キャプチャ対象を選択すると動画情報(MediaStream)を返り値として取得することができます。

![ポップアップの画像](/images/chrome-extension-article/screen-capture-popup.png)
_getCaptureStream()を実行した時に表示されるポップアップ( Chrome の場合 )_

次は、取得した動画情報(MediaStream)を仮想カメラとして使用するための処理を記述して行きます 🪁

# 画面キャプチャを仮想カメラとして使う

ブラウザからカメラなどのデバイス情報を取得するためには、[MediaDevices.getUserMedia()](https://developer.mozilla.org/ja/docs/Web/API/MediaDevices/getUserMedia) を使用します。

この関数は要求されたメディア情報を返す関数で、Web カメラの動画情報などは、この関数を通して取得します。そのため、この関数を上書きすることで、Web カメラなどの動画情報を別の動画情報に置き換えることができます。

今回の目的は、**画面キャプチャした動画情報を仮想カメラとして扱えるようにする**ことなので、この関数の挙動を変えれば目的を達成できそうですね！

ということで、上記で実装した`getCaptureStream()`の下に、以下の処理を記述して行きます 👇

```tsx:./src/main.ts
const getCaptureStream = async () => {/* -- 省略 -- */}

/**
 * @description 仮想カメラか判定する
 */
const isVirtualDevice = (video?: MediaTrackConstraints | boolean): boolean => {
  if (!video || video === true || !video.deviceId) return false;

  const deviceId = video.deviceId;

  if (Array.isArray(deviceId)) return deviceId.includes("virtual");
  if (typeof deviceId === "object") return deviceId.exact === "virtual";

  return deviceId === "virtual";
};

// 元々の`getUserMedia()`を保持しておく
const _getUserMedia = navigator.mediaDevices.getUserMedia.bind(
  navigator.mediaDevices
);

// `getUserMedia()`を上書きする
navigator.mediaDevices.getUserMedia = async function (
  constraints?: MediaStreamConstraints
) {
  // 仮想デバイスでなければ、元々のAPIを実行する
  if (!constraints || !isVirtualDevice(constraints.video)) {
    return _getUserMedia(constraints);
  }

  // 画面キャプチャのStream情報を取得する
  const stream = await getCaptureStream();

  // 仮想デバイスのStreamとして画面キャプチャのStreamを返す
  return stream;
}
```

やっていることは、特定の仮想デバイスを検出した時に、画面キャプチャの動画情報を返すようにしています。このように実装することで、あたかも画面キャプチャを映し出すデバイスがあるように見せかけています。

ここまで実装出来たら、あとは仮想デバイスを検出させるための処理を実装してきます 🔫

# 仮想デバイスを追加する

:::message
**`MediaDevices.enumerateDevices()` は、現在(2021/10/14)は実験的な機能です。**
:::

[Google Meet](https://apps.google.com/intl/ja/meet/) では、 [MediaDevices.enumerateDevices()](https://developer.mozilla.org/ja/docs/Web/API/MediaDevices/enumerateDevices) を使用して、通話に使用するデバイス(マイクやカメラなど)を取得しています(ﾀﾌﾞﾝ)。そのため、この関数の返り値を変更することで仮想デバイスを追加することができます。

先ほど実装した `getUserMedia()` の下に、仮想デバイスを追加する処理を実装しましょう 👇

```tsx:./src/main.ts
// `getUserMedia()`を上書きする
navigator.mediaDevices.getUserMedia = async function (
  constraints?: MediaStreamConstraints
) {
  /* -- 省略 -- */
}

// 元々の`enumerateDevices()`を保持しておく
const _enumerateDevices = navigator.mediaDevices.enumerateDevices.bind(
  navigator.mediaDevices
);

// `enumerateDevices()`を上書きする
navigator.mediaDevices.enumerateDevices = async function () {
  // 使用できるデバイス(マイク・カメラなど)を取得する
  const devices = await _enumerateDevices();

  // 仮想デバイスの情報を定義
  const virtualDevice = {
    groupId: "default",
    deviceId: "virtual",
    kind: "videoinput",
    label: "Screen Capture Virtual Camera 🎥",
  } as const;

  // 仮想デバイスを追加する
  devices.push({ ...virtualDevice, toJSON: () => ({ ...virtualDevice }) });

  return devices;
}
```

上記ように実装することで、カメラが無い環境でも [Google Meet](https://apps.google.com/intl/ja/meet/) が仮想カメラを使用可能なデバイスとして認識してくれます。

はい！これで Chrome 拡張の実装は完了です ✨
次は、実装したコードを拡張機能として認識させるための設定を行っていきます 🗿

# `manifest.json`を記述する

実装したソースコードを Chrome 拡張として使用にするには、`manifest.json` を書く必要があります。以下のようにして、拡張機能の権限や設定などを記述していきます 👇

```json:./public/manifest.json
{
  "version": "0.0.1",
  "name": "Screen Capture Virtual Camera 🎥",
  "description": "画面キャプチャを描画するための仮想カメラを提供します 🎥",
  "manifest_version": 2,

  // `content_scripts` の設定
  "content_scripts": [
    {
      // ビルドファイルを指定する
      "js": ["index.js"],

      // 拡張機能を使用できるサイトを記述
      "matches": [
        "http://localhost:*/*",
        "https://meet.google.com/*"
      ],

      // スクリプトの実行タイミングを指定
      "run_at": "document_start",

      // `js`で指定されたファイルを実行するようにする
      "all_frames": true
    }
  ],

  // chrome.extension.getURL()で取得できるようにする
  "web_accessible_resources": ["main.js"]
}
```

:::message
解説のために JSON 内にコメントを書いていますが、本番では削除しないと動かないので注意してください。
:::

設定ができましたら、次はビルドの設定を書いていきます 🏂

# Webpack を使ってビルドする

上記で実装したソースコードは、TypeScript なのでビルドする必要があります。以下のように `webpack.config.js` を定義して、ビルド設定を書いていきましょう 👇

```js:./webpack.config.js
const path = require("path");
const CopyPlugin = require("copy-webpack-plugin");

const isDev = process.env.NODE_ENV === "development";

/**
 * @type {import('webpack').Configuration}
 */
module.exports = {
  mode: isDev ? "development" : "production",

  devtool: isDev ? "inline-source-map" : false,

  entry: {
    // `contents`として扱うファイル
    index: path.resolve(__dirname, "./src/index.ts"),

    // `<script />`で挿入するファイル
    main: path.resolve(__dirname, "./src/main.ts"),
  },

  output: {
    clean: true, // ビルド時に前回のファイルがあった場合は削除する
    path: path.resolve(__dirname, "build"), // `./build`に結果を出力する
  },

  resolve: {
    extensions: [".js", ".ts"],
  },

  module: {
    rules: [
      {
        test: /\.ts$/,
        use: [
          {
            loader: "esbuild-loader", // esbuild-loaderを使ってビルド
            options: {
              loader: "ts",
              minify: !isDev,
              target: "es2015",
            },
          },
        ],
      },
    ],
  },

  plugins: [
    // `./public/manifest.json` を `./build/manifest.json` へコピーする
    new CopyPlugin({
      patterns: [{ from: "./public/manifest.json", to: "manifest.json" }],
    }),
  ],
};
```

設定ファイルが記述できたら、ビルドするためのコマンドを`package.json`に記述します 👇

```json:./package.json
{
  // ...

  "scripts": {
    "dev": "cross-env NODE_ENV=development webpack",  // 開発用
    "build": "cross-env NODE_ENV=production webpack", // 本番用
  },

  // ...
}
```

上記のコマンドを実行して、`./build` にビルド結果が出力されていれば大丈夫です 👌

:::message
必要なモジュールはインストール済みであることを想定しています。
:::

# 完成 ✨

これで拡張機能は完成です！

拡張機能の管理画面( chrome://extensions/ ) に移動して、ビルドして生成されたフォルダー(`./build`)を以下の画像のようにインストールします 👇

![作った拡張機能をインストールしている画像](/images/chrome-extension-article/chrome-extension-install.png)

インストールが完了して、以下のように拡張機能が表示されていれば大丈夫です 👌

![インストールした拡張が表示されている画像](/images/chrome-extension-article/chrome-extension-install-complete.png)

最後に、[Google Meet](https://apps.google.com/intl/ja/meet/) を開いて動作確認ができていれば、目的の挙動は達成です ✨

:::message alert
**Google Meet の挙動がおかしくなることがあります。**
**もし実行する際は、自己責任でお願いします 🙇‍♂️**
:::

![動作確認しているgif画像](/images/chrome-extension-article/chrome-extension-example.gif)
_右のキャプチャ内容が Google Meet のカメラ情報として表示されています_

ここまでお疲れさまでした 🙌

# 参考

拡張を作るにあたって、以下のサイトを参考にしました 🙇‍♂️

https://github.com/spite/virtual-webcam

https://techblog.securesky-tech.com/entry/2020/04/30/

https://qiita.com/massie_g/items/667627b6d12acc0163af

# あとがき

ここまで読んでくれてありがとうございます 🙏

実装する時間が無かったため、あまり作り込むことができず、まだまだ問題点などが多く残っていますが、最低限やりたいことはできたので、今回はこの辺りで終わろうと思います。

[@catnose](https://zenn.dev/catnose99)さんが作ってくれた [Emoji Live](https://live.catnose99.com/) は言わずもがな良いサービスで、**今回作った拡張なんかを使用しなくても十分使いやすモノです。** まだ使ったことのない方が居ましたら、ぜひ使いましょう！けっこう楽しいですよ？( 願わくは、私をカジュアル面談に誘って [Emoji Live](https://live.catnose99.com/) を使わせてください )

また、今回 Chrome 拡張を作るにあたって、[WebRTC](https://webrtc.org/) の深淵を少しだけ覗いてみましたが、 とても奥が深く、知れば知るほど面白い分野だと思いました。これが知れただけでも、良い経験になったと思います。しかし、なぜでしょうか。 Chrome 拡張を作ってからというもの、時折、深い深い闇の底から名状しがたい者達に呼ばれているような気がするのですが、、、たぶん気のせいでしょう。きっと。

記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。
これが誰かの参考になれば幸いです。

それではまた 👋
