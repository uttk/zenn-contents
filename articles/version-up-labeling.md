---
title: 'GitHub Actions を使ってラベルで package.json の version を更新する'
emoji: '🦡'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['githubactions']
published: true
publication_name: team_zenn
---

## この記事について

みなさん、こんにちは。
”狸” か ”穴熊” なら、私は ”貉” 派。uttk です。

リリースローの構築って、すごく悩みますよね。自動化しつつ柔軟性も持たせるとなると、要件にもよりますが、大体は妥協する感じになると思います。

特にバージョニングのところが個人的に苦労していて、前々から色々と試していたんですが、今回、 ﾁｮｯﾄｲｲ 感じのバージョニングフローを構築ができたので、この場を借りて共有していこうかなと思います 💪

ちなみに、今回紹介するバージョニングフローは以下のリポジトリで活用しています 👇

https://github.com/zenn-dev/zenn-vscode-extension

## バージョニングフローについて

今回作るバージョニングフローは以下のような感じになります 👇

```mermaid
flowchart TB
  subgraph actions[GitHub Actionsが起動]
    direction TB
    check[既に version が更新されてないかチェックする]
    -->|更新されていなければ| update[package.json の version を更新]
    --> commit[Pull Requestにコミットが追加される]
  end

  pr[リリース用の Pull Request を作成]
  --> labeling[対応するラベルを付与]
  --> actions
  --> merge[Pull Requestをマージする]
```

今回実装するのは、上記のフローの中の `GitHub Actionsが起動` の範囲の部分ですので、さっそく実装していきましょうー 🍛

## イベント対象の設定

GitHub Actions の workflow ファイルを `./.github/workflows/update-release-version.yaml` に実装していきます。

まずは workflow を発火させるための対象のブランチとイベントの種別を設定します。

今回は、対象ブランチを `main` にして、ラベルで workflow を動かしたいのでイベントの種別を `labeled`・`unlabeled` に設定します 👇

```diff yaml:/.github/workflows/update-release-version.yaml
+ name: ラベルに応じて package.json のバージョンを更新する
+ on:
+   pull_request:
+     branches:
+       - main
+     types:
+       - labeled
+       - unlabeled
```

## ラベルが無ければエラーを吐くようにする

次に、ラベルの貼り忘れ防止のためにラベルが Pull Request に貼られていなければエラーを吐くようにします。`on:` の下に以下のように実装します 👇

```diff yaml:/.github/workflows/update-release-version.yaml
 name: ラベルに応じて package.json のバージョンを更新する
 on:
   pull_request:
     branches:
       - main
     types:
       - labeled
       - unlabeled
+ jobs:
+   check_release_label:
+     runs-on: ubuntu-latest
+     steps:
+       - name: ラベルが無ければエラーを吐く
+         if: |
+           !contains(github.event.pull_request.labels.*.name, 'release:patch') &&
+           !contains(github.event.pull_request.labels.*.name, 'release:minor') &&
+           !contains(github.event.pull_request.labels.*.name, 'release:major')
+         run: |
+           echo "::error::リリースラベルを付与してください。labels: `release:patch`, `release:minor`, `release:major`"
+           exit 1
```

エラーを吐くようにする場合、分かりやすさのために `echo "::error::{message}"` で GitHub Actions のリザルト画面にエラーメッセージを出しておくといいと思います。

また、エラーメッセージの設定には他にもオプションがあるので、必要に応じて設定しておくといいかもです 👇

https://docs.github.com/ja/actions/using-workflows/workflow-commands-for-github-actions#setting-an-error-message

## package.json の version 変更を取得する

次に、package.json の version の変更を取得する処理を実装します。
目的としては、[package.json の version を更新する](#package.json-の-version-を更新する)タスクの判定処理で使用するためです。

```diff yaml:/.github/workflows/update-release-version.yaml
  # 省略
  jobs:
    check_release_label:
      # 省略
+   version_diff:
+    if: |
+      contains(github.event.pull_request.labels.*.name, 'release:patch') ||
+      contains(github.event.pull_request.labels.*.name, 'release:minor') ||
+      contains(github.event.pull_request.labels.*.name, 'release:major')
+    runs-on: ubuntu-latest
+    outputs:
+      chagned: ${{ steps.get_diff.outputs.changed }}
+    steps:
+      - uses: actions/checkout@v3
+
+      - name: マージ先を取得
+        run: git fetch origin ${{ github.base_ref }} --depth=1
+
+      - name: パージョンの変更を保持する
+        id: get_diff
+        run: echo "changed=$(git diff origin/${{ github.base_ref }} HEAD --relative "./package.json" | grep "^+.\+version" | wc -l)" >> $GITHUB_OUTPUT
```

処理内容としては、`release:patch`・`release:minor`・`release:major` のラベルが付いている時に、マージ先( 今回は main ブランチ )のコミットを取得し、`git diff` を用いて差分の行数を `outputs.changed` という変数に入れています。

変数定義の注意点として、`::set-output` や `::save-state` など定義方法は 2023/06 から使用できなくなります[^1]ので、`$GITHUB_OUTPUT` を使って定義する必要があります 👇

[^1]: https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/

https://docs.github.com/ja/actions/using-jobs/defining-outputs-for-jobs

## package.json の version を更新する

次に package.json の version を更新する処理を実装していきます 👇

```diff yaml:/.github/workflows/update-release-version.yaml
  # 省略
  jobs:
    check_release_label:
      # 省略
   version_diff:
      # 省略
+  update_version:
+    runs-on: ubuntu-latest
+    needs: [version_diff]
+    if: needs.version_diff.outputs.chagned == '0'
+    steps:
+      - uses: actions/checkout@v3
+        with:
+          ref: ${{ github.event.pull_request.head.ref }}
+
+      - uses: actions/setup-node@v3
+        with:
+          node-version: 16
+
+      - name: Gitの設定
+        if: steps.diff.outputs.changed == '0'
+        run: |
+          git config --global user.name 'github-actions[bot]'
+          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
+          git remote set-url origin https://github-actions:${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}
+        env:
+          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
+
+      - name: バージョンの更新(patch)
+        if: contains(github.event.pull_request.labels.*.name, 'release:patch')
+        run: pnpm version --patch --no-git-tag-version
+
+      - name: バージョンの更新(minor)
+        if: contains(github.event.pull_request.labels.*.name, 'release:minor')
+        run: pnpm version --minor --no-git-tag-version
+
+      - name: バージョンの更新(major)
+        if: contains(github.event.pull_request.labels.*.name, 'release:major')
+        run: pnpm version --major --no-git-tag-version
+
+      - name: 変更を PR にプッシュする
+        run: |
+          git add .
+          git commit -m "v$(grep version package.json | awk -F \" '{print $4}')"
+          git push origin HEAD
```

コードが長ったらしいですが、やっていることはラベルに対応するバージョン更新コマンド( `pnpm version` )を実行してコミット＆ Pull Request のブランチに push しているだけです。

また、[`version_diff:`](#package.json-の-version-変更を取得する) で取得した `outputs.changed` を使用して、バージョンの変更が無い場合にのみ処理を実行するようにしています 👇

```yaml:上記の実装から抜粋
if: steps.diff.outputs.changed == '0' # 変更がある場合は '1' 以上になる
```

これはラベルを付けなおした時などに二重にバージョンが更新されてしまうのを防ぐためです。

:::message
**もしラベルを付けなおした場合は、手動でバージョンを更新する必要があります。**
:::

## 完成 ✨

以上で実装は完了です！
main ブランチ向けに Pull Request を作って、その Pull Request にラベルを付けて正しくバージョンが更新されたコミットが Pull Request に追加されていれば OK👌 です。

また、今回実装したソースコードの全体を以下に載せておきますので、ご確認ください 👇

```yaml:/.github/workflows/update-release-version.yaml
name: ラベルに応じて package.json のバージョンを更新する
on:
  pull_request:
    branches:
      - main
    types:
      - labeled
      - unlabeled
jobs:
  check_release_label:
    runs-on: ubuntu-latest
    steps:
      - name: ラベルが無ければエラーを吐く
        if: |
          !contains(github.event.pull_request.labels.*.name, 'release:patch') &&
          !contains(github.event.pull_request.labels.*.name, 'release:minor') &&
          !contains(github.event.pull_request.labels.*.name, 'release:major')
        run: |
          echo "::error::リリースラベルを付与してください。labels: `release:patch`, `release:minor`, `release:major`"
          exit 1
  version_diff:
    if: |
      contains(github.event.pull_request.labels.*.name, 'release:patch') ||
      contains(github.event.pull_request.labels.*.name, 'release:minor') ||
      contains(github.event.pull_request.labels.*.name, 'release:major')
    runs-on: ubuntu-latest
    outputs:
      chagned: ${{ steps.get_diff.outputs.changed }}
    steps:
      - uses: actions/checkout@v3

      - name: マージ先を取得
        run: git fetch origin ${{ github.base_ref }} --depth=1

      - name: パージョンの変更を保持する
        id: get_diff
        run: echo "changed=$(git diff origin/${{ github.base_ref }} HEAD --relative "./package.json" | grep "^+.\+version" | wc -l)" >> $GITHUB_OUTPUT
  update_version:
    runs-on: ubuntu-latest
    needs: [version_diff]
    if: needs.version_diff.outputs.chagned == '0'
    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.event.pull_request.head.ref }}

      - uses: actions/setup-node@v3
        with:
          node-version: 16

      - name: Gitの設定
        if: steps.diff.outputs.changed == '0'
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
          git remote set-url origin https://github-actions:${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: バージョンの更新(patch)
        if: contains(github.event.pull_request.labels.*.name, 'release:patch')
        run: pnpm version --patch --no-git-tag-version

      - name: バージョンの更新(minor)
        if: contains(github.event.pull_request.labels.*.name, 'release:minor')
        run: pnpm version --minor --no-git-tag-version

      - name: バージョンの更新(major)
        if: contains(github.event.pull_request.labels.*.name, 'release:major')
        run: pnpm version --major --no-git-tag-version

      - name: 変更を PR にプッシュする
        run: |
          git add .
          git commit -m "v$(grep version package.json | awk -F \" '{print $4}')"
          git push origin HEAD
```

## あとがき

ここまで読んでくれてありがとうございます 🙏

今回の実装では、ラベルを付けなおした時の対応が自動化できてないので完全自動化とはいかないですが、リリース作業は慎重になるべきなので、ある程度は手作業する部分を残しておいてもいいかなと個人的に思っています。

ただもう少し改良の予知はあると思っているので、改良できたらまた紹介したいと思います 💪

記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。
これが誰かの参考になれば幸いです。

それではまた 👋
