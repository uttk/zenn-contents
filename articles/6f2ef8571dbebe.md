---
title: "NestJSでステージング環境を作る"
emoji: "🕹️"
type: "tech"
topics: ["typescript", "nestjs"]
published: true
---

# この記事について

[NestJS](https://nestjs.com/)でステージング環境を作っていきます💪

:::message
NestJSのひな型が既に作成されている事を想定しています
:::

# 必要なモジュールをインストール

```shell
$ npm i --save @nestjs/config cross-env
```

yarnを使っている場合は、以下のようになります。

```shell
$ yarn add @nestjs/config cross-env
```

- [@nestjs/config](https://docs.nestjs.com/techniques/configuration) の使い方やオプションについては、以下の記事を参照してください。

https://docs.nestjs.com/techniques/configuration

https://zenn.dev/uttk/scraps/5169d2bb2e3d767533f3#comment-65d278330ab6d7a86627

- [cross-env](https://www.npmjs.com/package/cross-env) は、環境変数を設定するために使用します。

:::message
📌 [cross-env](https://www.npmjs.com/package/cross-env) はメンテナンスモードに入りましたが、基本的な使い方をする分には何も問題はありません！
参考: https://github.com/kentcdodds/cross-env/issues/257
:::

# AppModuleにConfigModuleをインポートする

[@nestjs/config](https://docs.nestjs.com/techniques/configuration) を使うには、以下のように修正します。

```ts:app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
       // 他のクラスで、ConfigServiceを使えるようにする
      isGlobal: true,
      
      // ロードするenvファイルを指定する
      envFilePath: `.env.${process.env.NODE_ENV}`,

      // 複数指定する場合は、配列を使用する
      // 値に被りがある場合は、インデックスが小さいほうが優先されます
      // envFilePath: [`.env.${process.env.NODE_ENV}`, `.env.hoge`],
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`envFilePath`の部分で、`process.env.NODE_ENV`の値によって読み込むファイルを変更しています。これによって、ステージングが出来るようになります。

# npm scriptsを定義する

`nest`コマンドでひな型を生成した場合、`npm scripts`は以下のように定義されています。(※2020年12月現在)

```json:package.json
"scripts": {
    "prebuild": "rimraf dist",
    "build": "nest build",
    "format": "prettier --write \"src/**/*.ts\" \"test/**/*.ts\"",
    "start": "nest start",
    "start:dev": "nest start --watch",
    "start:debug": "nest start --debug --watch",
    "start:prod": "node dist/main",
    "lint": "eslint \"{src,apps,libs,test}/**/*.ts\" --fix",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:debug": "node --inspect-brk -r tsconfig-paths/register -r ts-node/register node_modules/.bin/jest --runInBand",
    "test:e2e": "jest --config ./test/jest-e2e.json"
},
```

上記の中で、`start`・`start:dev`・`start:debug`・`start:prod` の部分を [cross-env]() を使って以下のように修正します。

```diff:package.json
"scripts": {
    "prebuild": "rimraf dist",
    "build": "nest build",
    "format": "prettier --write \"src/**/*.ts\" \"test/**/*.ts\"",
+   "start": "cross-env NODE_ENV=development nest start",
+   "start:dev": "cross-env NODE_ENV=development nest start --watch",
+   "start:debug": "cross-env NODE_ENV=development nest start --debug --watch",
+   "start:prod": "cross-env NODE_ENV=production node dist/main",
    "lint": "eslint \"{src,apps,libs,test}/**/*.ts\" --fix",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:debug": "node --inspect-brk -r tsconfig-paths/register -r ts-node/register node_modules/.bin/jest --runInBand",
    "test:e2e": "jest --config ./test/jest-e2e.json"
},
```

# envファイルを定義

ここまで出来たら、あとは`env`ファイルを定義するだけです。

### 共通のファイル(常時読み込まれる)

`.env`ファイルは、`AppModule`**でインポートする時に何も設定しなくても読み込まれます。**
なので、複数の環境で共通の値などは `.env` ファイルに記述したほうがいいです。

:::message
値に被りがあった場合は、`envFilePath` で指定したファイルの値の方が優先されます。
:::

```env:.env(サンプル)
FILE_NAME=".env"
```

### NODE_ENV=developmentの時に読み込まれるファイル

development環境の時に使いたい変数を定義します。

```env:.env.development(サンプル)
FILE_NAME=".env.development"
```

**対応するnpm scripts**

```shell
$ npm run start
$ npm run start:dev
$ npm run start:debug
```

### NODE_ENV=productionの時に読み込まれるファイル

production環境の時に使いたい変数を定義します。

```env:.env.production(サンプル)
FILE_NAME=".env.production"
```

**対応するnpm scripts**

```shell
$ npm run start:prod
```

# 終わり

以上で環境構築は終わりです。お疲れ様でした🙌