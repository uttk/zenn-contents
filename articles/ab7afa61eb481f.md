---
title: "なぜWebpackの設定はTypeScriptで書けるのか？"
emoji: "📦"
type: "tech"
topics: ["nodejs", "typescript", "webpack"]
published: true
---

# この記事について

webpack の設定ファイルである`webpack.config.js`は、TypeScript で書いて Node.js 上で実行できます。しかし、**本来であれば TypeScript のソースコードは Node.js では実行できないはずです。** この事が気になった私は、今回その仕組みを調べてみたので、この場を借りてその調査結果を共有したいと思います 💪

**参照**

https://webpack.js.org/configuration/configuration-languages/

# 記事の概要

概要のみ知りたい人に向けて、以下にこの記事で解説する内容をまとめておきます 👇

- [webpack-cli](https://github.com/webpack/webpack-cli) では、 [rechoir](https://github.com/gulpjs/rechoir) を使って TypeScript を `require()` できるようにしているよ
- [rechoir](https://github.com/gulpjs/rechoir) は、 [ts-node](https://github.com/TypeStrong/ts-node) などを使って [require.extensions](https://nodejs.org/api/modules.html#modules_require_extensions) を拡張しているよ
- ちなみに、 [require.extensions](https://nodejs.org/api/modules.html#modules_require_extensions) は非推奨だよ
- webpack-cli が対応している言語は、 [node-interpret](https://github.com/gulpjs/interpret) から確認できるよ
- この記事の後半では、 [require.extensions](https://nodejs.org/api/modules.html#modules_require_extensions) を使って [YAML](https://ja.wikipedia.org/wiki/YAML) を `require()` できるようにするよ

# webpack-cli の実装を見る

まずは、[webpack-cli](https://github.com/webpack/webpack-cli)の実装を見てみましょう。
以下に、[設定ファイルを読み込む部分のソースコード](https://github.com/webpack/webpack-cli/blob/dbecc6750e3cd5ece92163545481ba3f608c48d0/packages/webpack-cli/lib/webpack-cli.js#L1504-L1550)を引用します。

```js:公式リポジトリから引用
const loadConfig = async (configPath) => {
  const { interpret } = this.utils;
  const ext = path.extname(configPath);
  const interpreted = Object.keys(interpret.jsVariants).find(
    (variant) => variant === ext
  );

  if (interpreted) {
    const { rechoir } = this.utils;

    try {
      rechoir.prepare(interpret.extensions, configPath);
    } catch (error) {
      if (error.failures) {
        this.logger.error(`Unable load '${configPath}'`);
        this.logger.error(error.message);

        error.failures.forEach((failure) => {
          this.logger.error(failure.error.message);
        });
        this.logger.error("Please install one of them");
        process.exit(2);
      }

      this.logger.error(error);
      process.exit(2);
    }
  }

  let options;

  try {
    options = await this.tryRequireThenImport(configPath, false);
  } catch (error) {
    this.logger.error(`Failed to load '${configPath}' config`);

    if (this.isValidationError(error)) {
      this.logger.error(error.message);
    } else {
      this.logger.error(error);
    }

    process.exit(2);
  }

  return { options, path: configPath };
};
```

なにやら色々な事をしていますが、上記のソースコードがやっている事をまとめると、

1. 設定ファイルをロードする( `require()`する )為の前処理
2. 設定ファイルのロード処理

に分けられます。

「 2. 設定ファイルのロード処理 」の方は、ファイルパスや拡張子を見て`require()`や`import()`を実行するだけなので、この記事では「1. 設定ファイルの前処理」のみ解説します。

以下に、そのソースコードを抜粋します 👇

```js:上記のソースコードから抜粋
const { interpret } = this.utils;
const ext = path.extname(configPath);
const interpreted = Object.keys(interpret.jsVariants).find(
  (variant) => variant === ext
);

if (interpreted) {
  const { rechoir } = this.utils;

  try {
    rechoir.prepare(interpret.extensions, configPath);
  } catch (error) {
    if (error.failures) {
      this.logger.error(`Unable load '${configPath}'`);
      this.logger.error(error.message);

      error.failures.forEach((failure) => {
        this.logger.error(failure.error.message);
      });
      this.logger.error("Please install one of them");
      process.exit(2);
    }

    this.logger.error(error);
    process.exit(2);
  }
}
```

まずは一行目を見てみましょう 👀

```js
const { interpret } = this.utils;
```

いきなり良く分からないモノが出てきましたが、これは [interpret](https://github.com/gulpjs/interpret) というライブラリになります。

このライブラリは、ファイル拡張子とそれに紐づいているローダーの情報を持ったオブジェクトを提供します。例えば、TypeScript の場合は、

```js:interpretのソースコードから抜粋
{
  '.ts': [
    'ts-node/register',
    'typescript-node/register',
    'typescript-register',
    'typescript-require',
    'sucrase/register/ts',
    {
      module: '@babel/register',
      register: function(hook) {
        hook({
          extensions: '.ts',
          rootMode: 'upward-optional',
          ignore: [ignoreNonBabelAndNodeModules],
        });
      },
    },
  ],
}
```

といった具合です。

2 行目以降の部分では設定ファイル( `webpack.config.*` )の拡張子を取得して、 その拡張子が [interpret](https://github.com/gulpjs/interpret) に定義されているかを判定しています。[^1]

[^1]: 具体的な情報は、[interpret のソースコード](https://github.com/gulpjs/interpret/blob/master/index.js)を参照してください

```js:interpretが設定ファイルの拡張子に対応しているかを判定している
const ext = path.extname(configPath);
const interpreted = Object.keys(interpret.jsVariants).find(
  (variant) => variant === ext
);
```

そして、拡張子が対応しているモノであれば ローダー情報を使って処理をしています。
以下の部分です 👇

```js:interpretのローダー情報を使って処理している部分
if (interpreted) {
  const { rechoir } = this.utils;

  try {
    rechoir.prepare(interpret.extensions, configPath);
  } catch (error) {
    /* -- 今回は関係ないので省略 -- */
  }
}
```

ソースコードを見ると、また `rechoir` と言う良く分からないモノが出てきましたね。しかし、これも [interpret](https://github.com/gulpjs/interpret) と同様にライブラリです。

この [rechoir](https://github.com/gulpjs/rechoir) は、渡されたローダー情報( `interpret.extensions` )を元に [require.extensions](https://nodejs.org/api/modules.html#modules_require_extensions) にローダーを設定します。

これによって、`.ts`ファイルなどの通常対応していない拡張子のファイルに対してローダーを適用することが出来るので、`webpack.config.ts`のようなファイルを`require()`することが可能となっています。

## まとめ

ここまでをまとめると、

1. [interpret](https://github.com/gulpjs/interpret) を使ってローダー情報を取得する
2. 設定ファイルの拡張子に [interpret](https://github.com/gulpjs/interpret) が対応しているか確認
3. 対応していれば、 [rechoir](https://github.com/gulpjs/rechoir) を使ってローダーを [require.extensions](https://nodejs.org/api/modules.html#modules_require_extensions) に登録する
4. 設定ファイル(`webpack.config.*`)をロードする

と言う流れになります。

これで、「 webpack の設定ファイルがなぜ TypeScript で書けるのか？ 」という謎は解決しました！

しかし、その謎を解決している`require.extensions`について解説していませんので、次の節でその詳細を見て行きましょう 👉

# require.extensions とは何か？

`require.extensions`は、特定の拡張子の処理方法を定義するためのモノです。この API を用いることで、通常では対応していない拡張子のファイルを Node.js 上で`require()`することができます。

今回の場合のように、TypeScript のソースコードを`require()`したい場合は、[ts-node](https://github.com/TypeStrong/ts-node) などのライブラリを使うことで簡単に実現できるようになります 👇

```js:.tsファイルをrequrie()できるようにする
require('ts-node/register');

// TypeScriptで書かれた設定情報を読み込む
const config = require('./config.ts');
```

結構便利な API ですが、実はこれは**非推奨**となっています。
理由が、[Node.js のドキュメント](https://nodejs.org/api/modules.html#modules_require_extensions)に書いてありましたので引用させてもらいます。

:::message alert
**Deprecated.** In the past, this list has been used to load non-JavaScript modules into Node.js by compiling them on-demand. However, in practice, there are much better ways to do this, such as loading modules via some other Node.js program, or compiling them to JavaScript ahead of time.
:::

要約すると、

「 処理が遅くなる可能性があるから、事前にコンパイルするとか他の方法で対応して！ 」

とのことです。まあ、わざわざランタイム上でコンパイルするなら、実行前にコンパイルしたほうが良いですよね。

公式が非推奨にしているモノなので、あまり多用しない方が良いと思いますが、こんな便利な API を知ったら、これを使って遊んでみたくなるのがプログラマーの性ってヤツですよね！

ということで、`require.extensions`を使って`.yaml`ファイルを`require()`できるようにしたいと思います！

# `.yaml`ファイルを`require()`できるようにする

まず必要なモジュールをインストールします。

```shell:yamlをインストール
$> npm i --save yaml
```

そしたら次に、`require.extensions`の処理を書いていきます 🖊

```js:./yaml-setup.js
const fs = require("fs")
const YAML = require('yaml')

// `.yaml`ファイルをrequire()できるようにする
require.extensions[".yaml"] = function (module, filename)  {
  const code = fs.readFileSync(filename, "utf8") // ファイルを読み込む

  module.exports = YAML.parse(code) // ファイルを解析してJSONとして返す
}
```

上記のファイルができたら、次は`require()`する`.yaml`ファイルと、`.yaml`ファイルを読み込む`index.js`ファイルを記述します。

```yaml:./target.yaml(requireするyamlファイル)
Hello:
  - World
```

```js:./index.js
const yamlData = require("./target.yaml") // 上記で定義したファイル

console.log(`出力結果:`, yamlData)
```

あとは、以下のように実行するだけです！

```shell:上記で定義したindex.jsを実行
$> node -r ./yaml-setup.js ./index.js

出力結果: { Hello: [ 'World' ] }
```

無事、`.yaml`ファイルの読み込みに成功しました 🎉

### `-r`オプションを使わない場合

また、以下のようにも書けます。

```js:./index.js
require("./yaml-setup") // `.yaml`ファイルを読み込めるようにする

const yamlData = require("./target.yaml") // 上記で定義したファイル

console.log(`出力結果:`, yamlData)
```

```shell
$> node ./index.js

出力結果: { Hello: [ 'World' ] }
```

# 参考リンク

https://qiita.com/erukiti/items/33fdf0170bbbb85c0f6c

https://nodejs.org/api/modules.html#modules_require_extensions

https://www.programmersought.com/article/50905381715/

# あとがき

ここまで読んでくれてありがとうございます 🙏

「 webpack の設定ファイルって、TypeScript で書けるのに Node.js で実行できるのはなんでだろう 🤔 」という疑問から調べてみましたが、想像とは違った形で実装されていて少し驚きました。

Module Hack の事や、なんとなく使っていた`-r`オプションの意味を理解できたり、色々なライブラリの事を知れたりと、小さな気づきから色々な事が学べる良い機会だったと思います。

みなさんも小さな気づきがあれば、調べてみると案外面白いことが分かるかもしれませんよ？

記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。
これが誰かの参考になれば幸いです。
それではまた 👋
