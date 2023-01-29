---
title: 'zenn-editor の Monorepo 環境を pnpm + Turborepo + lerna-lite で構築した話'
emoji: '✨'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [zenn, pnpm, turborepo, lernalite, monorepo]
published: true
publication_name: team_zenn
---

## この記事について

みなさん、こんちにちは。
好きな絶滅動物はアルゲンタヴィス。uttk です。

先日、zenn-editor の Monorepo 構成を Lerna から pnpm + Turborepo + lerna-lite に変えたので、この記事でその構成について解説していこうと思います 💪

https://github.com/zenn-dev/zenn-editor/pull/410

## なんで、その構成？

zenn-editor では Lerna を使って Monorepo 環境を構築していましたが、実際に Lerna を使っている部分はビルド時とリリース時のみで、ほとんどの機能を使用していませんでした。また、yarn workspace も併用していたため、違う Monorepo ツールが二つ存在していて、それぞれの役割が( 個人的に )分かりにくい状況でした。

そのため、Lerna を使わずに [pnpm が提供している workspace 機能](https://pnpm.io/ja/workspaces)に寄せる形で構築した結果、今回のような構成になりました。

## pnpm について

https://pnpm.io/ja/

pnpm は、npm や yarn と同じように node_modules を管理できるパッケージマネージャーです。

他のパッケージマネージャーと違い、シンボリックリンクを活用した独自の node_modules 構造を採用することで、肥大化しがちだった node_modules を軽量かつ安全に管理できる点などが画期的です。

パッケージマネージャーの中では最近出来たばかりなので歴史は浅いですが、もう既に多くのプロジェクトで使用されていて、個人的に今後日本でも使用されていきそうな気がします。

### 採用理由

今回の採用理由としては、動作が高速・workspace 機能ある・node_modules の構成が分かりやすいなどありますが、一番の決め手は npm や yarn に一抹の不安があったからです。

zenn-editor では yarn classic ( v1 ) を使っていたので、始めは yarn を使う事を検討しましたが、[yarn classic は既にメンテナンスモードに入っている](https://dev.to/arcanis/introducing-yarn-2-4eh1#what-will-happen-to-the-legacy-codebase)ため、今後の事を考えると yarn berry (v2 ~ v3)に移行する必要がありました。しかし、yarn berry では [Plug'n'Play](https://yarnpkg.com/features/pnp) や [Zero-Installs](https://yarnpkg.com/features/zero-installs) などのちょっと癖の強い独自機能が盛り込まれているため、使いこなすには結構な労力が必要そうな感じがしたので、断念しました。

次に npm ですが、こちらは動作がまだまだ遅かったり、hoist (巻き上げ) に対するオプションが無かったり、 少し前にレイオフなどが発生して開発に影響が出ている[^1]などの点から断念しました。バックに GitHub がいるとはいえ、扱いづらさが少し目立つ印象です。

[^1]: 実際 [npm のブログ](https://github.blog/changelog/2022-10-24-npm-v9-0-0-released/)を見ると、新しい機能の開発よりも既存の機能のブラッシュアップに重きを置いている感じがします。

pnpm も不安要素は大いにありますが、[他のパッケージマネージャーが pnpm の node_modules 構成に対応している](https://pnpm.io/ja/blog/2021/12/29/yearly-update#競合ツールの動向)辺り、今後に期待が出来そうなので、これからガンガン使って行こうかと思います 💪

## Turborepo について

https://turbo.build/repo

Turborepo は Vercel 社が開発しているタスク実行ツールです。

Monorepo 内の npm scripts などを依存関係を考慮して実行してくれますし、タスクの結果をキャッシュしてより高速にタスクを完了できるようにしてくれます。

### 採用理由

導入が簡単かつ使ってみて良さそうだったので、フィーリングで雑に導入してみました 😎

Lerna を使っていた時よりも npm scripts の実行が早くなって開発体験がすごく向上したので、入れるなら入れてみてもいいじゃないかと個人的に思います。実際、導入もそこまで難しくないですし、やっていることが限定的なので、使わなくなったとしても別のツールに移行しやすいと思います。

## lerna-lite について

https://github.com/lerna-lite/lerna-lite

Lerna-Lite は、Lerna から主に `version` と `publish` の機能を抜き出したライブラリです。

ほとんど Lerna と同じように使えるので、Lerna を使っていた Monorepo 環境であればすぐに使うことができます。また、pnpm の workspaces にも対応しているので導入しやすいです。

### 採用理由

zenn-editor では Lerna の `publish` 機能を使って packages 内のパッケージをリリースしていたので、Lerna を使用をやめるにあたってリリースフローを再構築する必要がありました。

しかし、Lerna 以外に良いリリースフローがあまり見当たらなかったため、Lerna-Lite を使ってリリースフロー自体は変更無いように構築しました。そのおかげで、リリースはこれまで通りできますし、Lerna よも機能が絞られていてより役割が明確になった気がします。

## あとがき

はい、以上で移行の解説は終わりです。

調べてみると、多くの場所で解説されている Monorepo はどれも大規模なプロジェクトが想定されていて、zenn-editor のユースケースと若干のズレがありました。

そこで、今回の Monorepo ツールをなるべく使わない構成にしたのですが、これが現状のベストプラクティスという訳ではなく、あくまで良さそうなモノを組み合わせただけに過ぎず、今後の運用で色々と知見を得られたらと思っています。

ここまで読んでいただいてありがとうございます 🙏
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。
これが誰かの参考になれば幸いです。

それではまた 👋
