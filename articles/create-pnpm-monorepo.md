---
title: 'もはや pnpm と Turborepo で Monorepo 環境作れるから'
emoji: '🌿'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [pnpm, turborepo, monorepo]
published: true
---

## この記事について

みなさん、こんにちは。
先日、pnpm + Turborepo + lerna-lite で作った Monorepo 環境の解説記事を書きました 👇

https://zenn.dev/team_zenn/articles/new-monorepo-env

今回は簡易的な Monorepo 環境を作って上記の構成を解説して行こうかと思います 💪 ( 最低限の Monorepo 機能しかないので需要はあるかは分かりませんが... )

:::message
注意として、pnpm や node.js などのインストールは済ませてあることを前提に解説してきます。あらかじめご了承ください 🙏
:::

では、さっそくやっていきましょうー 🍫

## ディレクトリ構造について

今回の作る Monorepo 環境の最終的な全体のディレクトリ構造は以下のようになります 👇

```:関係があるファイルのみ記述
.
├── packages/
│   ├── lib-a/
│   |   ├── src/
│   |   |   └──index.ts
│   │   └── package.json
│   └── lib-b/
│       ├── src/
│       |   └──index.ts
│       └── package.json
├── package.json
├── pnpm-workspace.yaml
└── turbo.json
```

また、環境のバージョン関係は以下のようになります 👇

| name    | version  |
| ------- | -------- |
| node.js | v22.13.1 |
| pnpm    | v10.1.0  |

上記が確認できましたら、最初はルートにある package.json の設定をしていきましょうー 🍏

## package.json の設定

以下のコマンドを実行して package.json を作成します 👇

```shell:./package.jsonを作成する
$> pnpm init
```

```diff:上記のコマンドを実行すると package.json が生成されます
  .
+ └── package.json
```

次に、生成された `./package.json` を以下のように編集します 👇

```diff json:./package.json
  {
    "name": "monorepo-example",
    "version": "1.0.0",
    "description": "",
    "scripts": {
      "test": "echo \"Error: no test specified\" && exit 1"
    },
    "keywords": [],
    "author": "",
    "license": "MIT",
+   "packageManager": "pnpm@10.1.0",
+   "engines": {
+     "pnpm": ">=10.1.0"
+   }
  }
```

`"packageManager"` は後述する turborepo を実行する時に必要な設定です。

`"engines"` の方は設定しておくと、想定していないバージョンで実行してしまった時にエラーを出すようにできます。今回は pnpm v10.1.0 以上を指定しています。

一応、以下のように `"npm"` や` "yarn"` などに `"use pnpm please!"` のような値を設定しておくと、間違って yarn や npm を実行できないようにできます 👇

```diff json:./package.json
  "engines": {
+   "npm": "use pnpm please!",
+   "yarn": "use pnpm please!",
    "pnpm": ">=10.1.0"
  }
```

しかし、配布するパッケージの package.json に書いてしまうとパッケージを使うユーザー側でエラーになってしまったり、npm の場合は `engine-strict=true` 設定が必要だったりと使い勝手が悪いので、今回は設定しないようにしています。

:::message
もし設定する場合は、ワークスペースのルートにある `package.json` か、`"private": true` が設定されているパッケージにのみ設定してください 👮‍♀️
:::

## pnpm workspace の設定ファイルを作成する

次は workspace の設定をします。
pnpm の場合、`./pnpm-workspace.yaml` にワークスペースに含むまたは除外するディレクトリーを glob パターンで指定するだけで設定できます 👇

```yaml:./pnpm-workspace.yaml
packages:
  - 'packages/*'
```

上記の設定で `packages` 直下のサブディレクトリが workspace の対象として扱えるようになりました！

次は実際に `packages` にパッケージを実装していきましょうー 🎒

## packages/lib-a を作成する

依存される側のパッケージを定義します。
まずはディレクトリと必要なパッケージをインストールします 👇

```shell:
# lib-a のディレクトリを作成
$> mkdir -p ./packages/lib-a
$> cd ./packages/lib-a

# package.json を作成
$> pnpm init

# typescript をインストール
$> pnpm add -D typescript
```

次に、ソースコードを実装します。依存される側なので、簡単な変数のみ実装します 👇

```js:./pacakages/lib-a/src/index.ts
export const message = 'HELLO WORLD!'
```

次に、package.json を以下のように記述します 👇

```diff json:./packages/lib-a/package.json
  {
+   "name": "@uttk/lib-a",
    "version": "1.0.0",
+   "main": "./dist/index.js",
+   "scripts": {
+     "build": "tsc ./src/index.ts --outDir ./dist --declaration"
+   },
    "devDependencies": {
      "typescript": "^5.7.3"
    }
  }
```

:::message
パッケージ名は間違って変なパッケージをインストールしないように `@uttk` をプレフィックスで付けています。
:::


ここまで記述できたら、以下のコマンドを実行してビルドファイルが正しく出力されていれば OK👌 です。

```shell:lib-a の build タスクを実行
$> cd ./packages/lib-a
$> pnpm build

> @uttk/lib-a@1.0.0 build /monorepo-example/packages/lib-a
> tsc ./src/index.ts --outDir ./dist --declaration

# ルートに居る状態で以下のコマンド実行しても同じように出来ます
# $> pnpm --filter @uttk/lib-a build
```

これで lib-a パッケージの実装は完了です。
次は lib-a に依存するパッケージである lib-b を実装していきましょうー 🥬

## packages/lib-b を作成する

まずは lib-b のディレクトリと package.json を作成します 👇

```shell
# lib-b のディレクトリを作成
$> mkdir -p ./packages/lib-b
$> cd ./packages/lib-b

# package.json を作成
$> pnpm init

# typescript をインストール
$> pnpm add -D typescript
```

次に、lib-b は lib-a に依存するので以下のコマンドで lib-a をインストールします 👇

```shell:lib-b に lib-a をインストールする
$> cd ./packages/lib-b
$> pnpm add '@uttk/lib-a@workspace:*'

# ルートに居る状態で以下のコマンド実行しても同じように出来ます
# $> pnpm --filter lib-b add '@uttk/lib-a@workspace:*'
```

上記のコマンドが成功したら、次は package.json を以下のように修正します 👇

```diff json:./packages/lib-b/package.json
  {
+   "name": "@uttk/lib-b",
    "version": "1.0.0",
+   "main": "./dist/index.js",
+   "scripts": {
+      "build": "tsc ./src/index.ts --outDir ./dist --declaration"
+    },
    "devDependencies": {
      "typescript": "^5.7.3"
    },
    "dependencies": {
      "@uttk/lib-a": "workspace:*"
    }
  }
```

`"main"` や `"build"` は lib-a の時と同じですが、`"dependencies"` 内の lib-a のバージョンを `"workspace:*"`に修正しています。これによって、常にローカルの lib-a の方を参照するようになります。( ※ もしこの挙動が嫌な場合は、適切なバージョンを設定してください。 )

次に `src/index.ts` を以下のように実装します 👇

```ts:./packages/lib-b/src/index.ts
import { message } from "@uttk/lib-a";

console.log(`${message} from lib-b`)
```

ここまで実装できたら、build タスクを実行して正しくビルドできたら OK👌 です。

```shell:lib-b の build タスクを実行
$> cd ./packages/lib-b
$> pnpm build

> @uttk/lib-b@1.0.0 build /monorepo-example/packages/lib-a
> tsc ./src/index.ts --outDir ./dist --declaration

# ルートに居る状態で以下のコマンド実行しても同じように出来ます
# $> pnpm --filter @uttk/lib-b build
```

ビルドが成功したら、ビルドファイルを実行してみてテキストが正しく表示されていれば lib-b の完成です！

```shell:ビルドファイルを実行する
$> node ./dist/index.js

HELLO WORLD! from lib-b
```

ここまで出来れば、workspace の設定＆作成は完了です。
次はより workspace 内のタスクを扱いやすくするための設定を行っていきます 🎯

## Turborepo でタスクを実行する

Turborepo は Monorepo 内のタスクを依存関係を考慮して実行してくれるツールです。

https://turbo.build/repo

今回は `./packages` 内の workspace の `build` コマンドを一括で実行できるようにしたいと思います。

まずは、Turborepo をルートにインストールします 👇

```shell
# ルートにインストールする場合は`-w, --workspace-root`が必要です
$> pnpm add -w -D turbo
```

インストールが完了したら、次は `./turbo.json` を以下のように記述します 👇

```json:./turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    }
  }
}
```

`"dependsOn"` に `"^build"` を設定することで、依存関係を考慮して `build` タスクを実行してくれます。( 今回の場合は `lib-a の build` → `lib-b の build` の順番で実行してくれます )

`"outputs"` にビルド結果を含むディレクトリやファイルのパスを glob パターンで設定すると、ファイルの差分を考慮してタスクを実行してくれます。これにより、変更の無いパッケージを無駄にビルドしなくて済むようになります。

これで Turborepo の設定は完了です。
実行しやすいように package.json の `"scripts"` に `"build"` を追加します 👇

```diff json:./package.json
  "scripts": {
+   "build": "turbo build",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```

追加できたら以下のコマンドを実行して正しくビルドできれば OK👌 です。

```shell:pnpm build を実行してみる
$> pnpm build

> monorepo-example@1.0.0 build /monorepo-example
> turbo build

• Packages in scope: lib-a, lib-b
• Running build in 2 packages
• Remote caching disabled
lib-a:build: cache miss, executing 5107e54a8a529999
lib-a:build:
lib-a:build: > @uttk/lib-a@1.0.0 build /monorepo-example/packages/lib-a
lib-a:build: > tsc ./src/index.ts --outDir ./dist --declaration
lib-a:build:
lib-b:build: cache miss, executing 87efcd3afd8b858c
lib-b:build:
lib-b:build: > @uttk/lib-b@1.0.0 build /monorepo-example/packages/lib-b
lib-b:build: > tsc ./src/index.ts --outDir ./dist --declaration
lib-b:build:

 Tasks:    2 successful, 2 total
Cached:    0 cached, 2 total
  Time:    3.942s
```

## 完成 ✨

はい、これで今回の Monorepo 環境構築は完了です。お疲れさまでした！

基本的には、pnpm workspace と Turobrepo の設定だけなので、そこまで難しくは無かったと思いますが、扱うパッケージによってはより複雑な設定が必要になるかもしれません。その際は、Turborepo の便利なオプションが色々ありますので、確認しておくと良いかと思います 👇

https://turbo.build/repo/docs/reference/configuration

また、pnpm の `--filter` オプションを活用すると、パッケージのコマンドをルートから実行できたりして開発が捗ると思いますので、こちらも確認しておくと良いかと思います 👇

https://pnpm.io/ja/filtering#--filter-package_name

### リリースフローについて

リリースフローについては、プロジェクトによって変わってきますので、ここでは解説しませんが、一例として [zenn-editor](https://github.com/zenn-dev/zenn-editor) では Lerna-Lite を使用しています 👇

https://github.com/lerna-lite/lerna-lite

また、柔軟なリリースフローを構築できるツールとして Changesets などがありますので、そちらも検討してみてもいいかもしれません 👇

https://github.com/changesets/changesets

## あとがき

ここまで読んでくれてありがとうございます 🙏

Monorepo の環境構築というと Lerna などのようなツールを使うことが多いと思いますが、今ではパッケージマネージャーに備わっている workspace 機能と、それを補助するツールを使うだけでもある程度の Monorepo 環境は作れると思いますので、この機会に是非やってみてはいかがでしょうか。

私は zenn-editor で今回の構成をやってみたので、己の Monorepo 力を磨きながら運用していこうかと思います。( もし負債になったら責任もって私が返済する覚悟 😇 )

ここまで読んでいただいてありがとうございます 🙏
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。
これが誰かの参考になれば幸いです。

それではまた 👋
