---
title: Zenn の VSCode 拡張（ 簡易版 ）を作って VSCode Web Extension に入門してみよう！
emoji: 🥳
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [zenn, vscode]
published: false
publication_name: team_zenn
---

# 💬 この記事について

どうも皆さん、こんにちは。
Zenn でお手伝いさせてもらっています uttk です。

今回は、先日公開された [github.dev](https://github.dev) で Zenn のコンテンツを表示するための VSCode 拡張（β）の簡易版を作って VSCode Web Extension に入門してみようと思います 💪

https://info.zenn.dev/release-vscode-extension

:::message
全ての機能を作ると冗長になってしまうため、この記事では記事コンテンツのみを扱う機能に絞って実装します。あらかじめご留意ください 🙏
:::

あと、参考までに使用する環境は以下の通りです 👇

| 名前    | バージョン |
| ------- | ---------- |
| Node.js | v18.12.1   |
| yarn    | 1.22.19    |

さてさて、上記が確認できましたらさっそく入門していきましょうー 🥨

# 📂 VSCode 拡張のテンプレートを作成

まず初めに、以下のコマンドを実行して VSCode 拡張を作るためのテンプレートを作成します 👇

```shell
# 適当なフォルダーに入る
$> cd example

# ローカルに `yo` と `generator-code` をインストール
$> yarn init -y && yarn add yo generator-code

# テンプレートを作成
$> yarn yo code
```

↑ のコマンドを実行すると質問されるので以下のように答えます。

```bash
# ? What type of extension do you want to create?
> New Web Extension (TypeScript)

# ? Whats the name of your extension?
> Zenn VSCode Extension Example

# ? What's the identifier of your extension?
> zenn-vscode-extension-example # Enterをそのまま押しても大丈夫 🙆‍♂️

# ? What's the description of your extension?
> Zennの記事コンテンツを表示するVSCode拡張

# ? Initialize a git repository?
> Y  # Enterをそのまま押しても大丈夫 🙆‍♂️

# ? Which package manager to use?
> yarn # ここでは yarn としてますが、使っているモノを選択してください
```

上記のコマンドを実行して、ファイルが作成され必要なパッケージがインストールされていれば大丈夫です 👌

## ブラウザで拡張を実行する

次に生成されたフォルダーに入って、以下のコマンドを実行することでブラウザ上で拡張機能を実行することができます 👇

```shell
$> yarn run-in-browser
```

![test](/images/create-zenn-vscode-extension/open-in-browser.png)
_`yarn run-in-browser`を実行したときに開くブラウザ_

## VSCode 上で拡張を実行する

VSCode を使っている場合は、VSCode のデバッグ機能を使うことで拡張を実行できます 👇

![](/images/create-zenn-vscode-extension/run-web-extension.png)
_デバッグパネルで`Run Web Extension`を選択する_

`F5`キーまたはデバッグ実行ボタンを押してデバッグを開始すると、新しい VSCode ウィンドウが立ち上がります 👇

![](/images/create-zenn-vscode-extension/new-vscode-window.png)
_新しい VSCode ウィンドウで拡張を実行できます_

### ブレークポイントを打つ

デバッグモードで拡張を実行している場合は、**拡張のソースコードを開いている VSCode のウィンドウ**内でブレークポイントを打つことができます 👇

![](/images/create-zenn-vscode-extension/breakpoint.png)
_実行を停止したい所にブレークポイントを打つ_

ブレークポイントを打ったあと、拡張がインストールされている側の VSCode ウィンドウで Dev Tools を開きます。

![](/images/create-zenn-vscode-extension/toggle-dev-tools.png)
_`>Developer: Toggle Developer Tools`で開くことができます_

この状態で処理を実行すると、ブレークポイントを打った場所が実行された時に処理が停止します。

以上で、ひな形の作成と基本的なデバッグ手順は終了です！
次は実際に VSCode 拡張作っていきましょー 🛬

# 📝 記事作成コマンドを実装する

まずは記事ファイルを作成するコマンドから作っていきます。`package.json` を以下のように修正しましょう 👇

```diff json:./package.json
  {
    /* -- 省略 -- */
    "activationEvents": [
-     "onCommand:zenn-vscode-extension-example.helloWorld"
+     "*"
    ],
    "browser": "./dist/web/extension.js",
    "contributes": {
      "commands": [
        {
-         "command": "zenn-vscode-extension-example.helloWorld",
-         "title": "Hello World",
+         "command": "zenn-vscode-extension-example.new-article",
+         "title": "Zenn: New Article"
        }
      ],
      /* -- 省略 -- */
    },
    /* -- 省略 -- */
  }
```

次に記事作成処理を実装していきますが、この記事では本元の Zenn の VSCode 拡張と同じようなフォルダー構造にします 👇

```shell:関係のあるファイルのみ記述
.
├─ src
│  └─ web
│     ├─ commands
│     │  └─ newArticle.ts # 記事作成コマンドの処理を実装するファイル
│     ├─ context
│     │  └─ commands.ts # コマンドの初期化処理を行うファイル
│     └─ extension.ts # 拡張の起点となるファイル
└─ package.json
```

## extension.ts を実装する

まずは `./src/web/extension.ts` を以下のように実装します 👇

```ts:./src/web/extension.ts
import * as vscode from "vscode";
import { initializeCommands } from "./context/commands"; // この後実装します

/** 拡張内の共通の情報をまとめたオブジェクト */
export interface AppContext {
  extension: vscode.ExtensionContext
  articlesFolderUri: vscode.Uri; // 記事コンテンツを格納しているフォルダーへのUri
}

// 拡張がアクティベートされる時に実行される関数
export function activate(extensionContext: vscode.ExtensionContext) {
  const workspaceUri = vscode.workspace.workspaceFolders?.[0].uri;

  if (!workspaceUri) {
    return vscode.window.showErrorMessage("ワークスペースがありません");
  }

  const context: AppContext = {
    extension: extensionContext,
    articlesFolderUri: vscode.Uri.joinPath(workspaceUri, "articles"),
  };

  extensionContext.subscriptions.push(
    ...initializeCommands(context) // コマンドの初期化処理
  );
}

// This method is called when your extension is deactivated
export function deactivate() {}
```

`extension.ts` は拡張の起点となるファイルなので、簡単な登録処理のみ記述するようにしています。

またポイントとして、`AppContext` を作って拡張内で共通に使いたいデータなどをまとめています。こうすることによって、テストしやすくなりますし、なるべくグローバル変数を使用しないでデータのやり取りができます。( まあ、バケツリレー的な感じにはなりやすいですが、致し方ない 😿 )

## context/commands.ts を実装する

次に `./src/web/context/commands.ts` を実装します。
このファイルではコマンド関連の初期化処理を行いますので、以下のように実装しましょう 👇

```ts:./src/web/context/commands.ts
import * as vscode from "vscode";
import { AppContext } from "../extension";
import { newArticleCommand } from "../commands/newArticle"; // この後実装します

export const initializeCommands = (context: AppContext): vscode.Disposable[] => {
  return [
    // 記事ファイルの作成コマンド
    vscode.commands.registerCommand(
      "zenn-vscode-extension-example.new-article",
      newArticleCommand(context)
    ),
  ];
};
```

ここの処理でコマンド ID と実際の処理の紐づけを行っています。
注意点として、`vscode.commands.registerCommand()` の第一引数は package.json で定義した文字列を指定する必要がありますので注意してください！よくタイポして動かなくなることがあるので、ホント気をつけて下さいね... ( n 敗 )

ここまで実装したら、あとは記事の作成処理を書くだけです！
`./src/web/commands/newArticle.ts` に処理を実装していきましょうー 📮

## commands/newArticle.ts を実装する

次に、記事作成コマンドの具体的な処理を実装してきます 👇

```ts:./src/web/commands/newArticle.ts
import * as vscode from "vscode";
import { AppContext } from "../extension";

/** 14文字のランダムな文字列を返す */
export const generateSlug = (): string => {
  const a = Math.random().toString(16).substring(2);
  const b = Math.random().toString(16).substring(2);
  return `${a}${b}`.slice(0, 14);
};

/** ランダムに Emoji を返す */
export const pickRandomEmoji = (): string => {
  const emojiList =["😺","📘","📚","📑","😊","😎","👻","🤖","😸","😽","💨","💬","💭","👋", "👌","👏","🙌","🙆","🐕","🐈","🦁","🐷","🦔","🐥","🐡","🐙","🍣","🕌","🌟","🔥","🌊","🎃","✨","🎉","⛳","🔖","📝","🗂","📌"]; // prettier-ignore
  return emojiList[Math.floor(Math.random() * emojiList.length)];
};

/** 記事のテンプレート文字列を生成する関数 */
const generateArticleTemplate = () =>
  [
    "---",
    `title: ""`,
    `emoji: "${pickRandomEmoji()}"`,
    `type: "tech" # tech: 技術記事 / idea: アイデア`,
    "topics: []",
    `published: false`,
    "---",
  ].join("\n") + "\n";

/** 記事の新規作成コマンドの実装 */
export const newArticleCommand = (context: AppContext) => {
  return async () => {
    const { articlesFolderUri } = context;

    // 記事のテンプレート文字列を作成
    const text = new TextEncoder().encode(generateArticleTemplate());

    // 記事の保存先のUriを作成
    const aritcleSlug = generateSlug()
    const fileUri = vscode.Uri.joinPath(articlesFolderUri, `${aritcleSlug}.md`)

    // ファイルを作成
    await vscode.workspace.fs.writeFile(fileUri, text);

    vscode.window.showInformationMessage("記事を作成しました");
  };
};
```

`vscode.workspace.fs.writeFile()` に渡させる出力値は、`Uint8Array`しかないので変換する必要がありますが、実はこの拡張はブラウザ上でも動くことになっているので、**ブラウザには無い `Buffer` ではなく `TextEncoder` で変換する必要があります。**

また以下のようにテンプレート文字列を配列で生成できるようにしておくと、オプションによって表示内容を切り替えやすくなるのでオススメです 👇

```ts:記事のテンプレート文字列を配列で定義している様子
/** 記事のテンプレート文字列を生成する関数 */
const generateArticleTemplate = () =>
  [
    "---",
    `title: ""`,
    `emoji: "${pickRandomEmoji()}"`,
    `type: "tech" # tech: 技術記事 / idea: アイデア`,
    "topics: []",
    `published: false`,
    "---",
  ].join("\n") + "\n";

```

はい！ここまででコマンドの実装は終了です！
実際に今実装したコマンドを実行してみましょうー 🚀 ﾜｰｲ

## コマンドを実行してみる

![](/images/create-zenn-vscode-extension/execute-extension.gif)
_コマンドパレットから`New Zenn Article`コマンドを実行する様子_

上の画像のように、新しく記事ファイルが作成されていればオッケー 👌 です！
次は、TreeView の実装にいきましょうー 🐢

# 🐿 TreeView に記事一覧を表示

次に [TreeView](https://code.visualstudio.com/api/extension-guides/tree-view) に記事一覧を表示できるようにします。package.json を以下のように修正しましょう 👇

```diff json:./package.json
  {
    /* -- 省略 -- */
    "contributes": {
      "commands": [
        /* -- 省略 -- */
      ],
+     "viewsContainers": {
+       "activitybar": [
+         {
+           "id": "zenn-treeview",
+           "title": "Zenn TreeView",
+           "icon": "$(window)"
+         }
+       ]
+     },
+     "views": {
+       "zenn-treeview": [
+         {
+           "id": "zenn-articles-treeview",
+           "name": "Articles"
+         }
+       ]
+     }
    },
    /* -- 省略 -- */
  }
```

この設定によって、拡張機能がアクティベートされるとサイドバーに `"icon"` で設定したアイコンが表示されます。ちなみに今回は VSCode に標準搭載されているアイコン( `$(window)` )を使っています 👇

https://code.visualstudio.com/api/references/icons-in-labels

上記の設定ができたら、次は以下のフォルダー構造に従って TreeView を実装していきましょうー ⛱

```diff shell:関係のあるファイルのみ記述
  .
  ├─ src
  │  └─ web
  │     ├─ commands
  │     │  └─ newArticle.ts # 記事作成コマンドの処理を実装するファイル
+ │     ├─ schemas
+ │     │  └─ article.ts # 記事情報を扱う処理を実装するファイル
+ │     ├─ treeview
+ │     │  ├─ articlesTreeViewProvider.ts # 記事のTreeViewProviderを実装するファイル
+ │     │  └─ articleTreeItem.ts # 記事のTreeItemを実装するファイル
  │     ├─ context
  │     |  ├─ commands.ts
+ │     │  └─ treeview.ts # TreeViewの初期化処理を行うファイル
  │     └─ extension.ts
  └─ package.json
```

## extension.ts に `initiallizeTreeview()` を追加

まずは、`src/web/extension.ts` に処理を追加します 👇

```diff ts:./src/web/extension.ts
  import * as vscode from "vscode";
  import { initializeCommands } from "./context/commands";
+ import { initializeTreeView } from "./context/treeview"; // この後実装します

  /* -- 省略 -- */

  // 拡張がアクティベートされる時に実行される関数
  export function activate(extensionContext: vscode.ExtensionContext) {
    /* -- 省略 -- */

    context.subscriptions.push(
      ...initializeCommands(context), // コマンドの初期化処理
+     ...initializeTreeView(context), // TreeViewの初期化処理
    );
  }

  /* -- 省略 -- */
```

コマンド実装の時と同様に初期化処理を登録するだけです。
次は `initializeTreeView()` を実装しましょうー 🏗

## context/treeview.ts を実装する

TreeView の登録処理を以下のように実装します 👇

```ts:./src/web/context/treeview.ts
import * as vscode from "vscode";
import { AppContext } from "../extension";
import { ArticlesTreeViewProvider } from "../treeview/articlesTreeViewProvider"; // この後実装します

export const initializeTreeView = (context: AppContext): vscode.Disposable[] => {
  const articlesTreeViewProvider = new ArticlesTreeViewProvider(context);

  return [
    // 記事のTreeViewを登録する
    vscode.window.createTreeView("zenn-articles-treeview", {
      treeDataProvider: articlesTreeViewProvider,
    }),
  ];
};
```

`ArticlesTreeViewProvider` は、記事の TreeView を提供するためのクラスです。 このクラスを `vscode.window.createTreeView()` を使って登録することで、TreeView を表示できます。またコマンドの時もそうでしたが、`vscode.window.createTreeView()` の第一引数には package.json で設定した値を指定する必要があるので注意してください！( n 敗 )

## treeview/articlesTreeViewProvider.ts を実装する

次に TreeView を提供するためのクラスである `ArticleTreeViewProvider` を実装します 👇

```ts:./src/web/treeview/articlesTreeViewProvider.ts
import * as vscode from "vscode";
import { AppContext } from "../extension";
import { ArticleTreeItem } from "./articleTreeItem"; // この後実装します
import { getArticleContents, ArticleContentError } from "../schemas/article"; // この後実装します

type TreeDataProvider = vscode.TreeDataProvider<vscode.TreeItem>;

export class ArticlesTreeViewProvider implements TreeDataProvider {
  private readonly context: AppContext;

  constructor(context: AppContext) {
    this.context = context;
  }

  // 引数の element には getChildren() で返した配列の要素が渡されます
  async getTreeItem(element: vscode.TreeItem): Promise<vscode.TreeItem> {
    return element;
  }

  // 表示したい TreeItem の配列を返します
  async getChildren(element?: vscode.TreeItem): Promise<vscode.TreeItem[]> {
    if (element) return [element];

    // 記事一覧情報を取得する
    const articleContents = await getArticleContents(this.context);

    // 表示する TreeItem の配列を作成
    const treeItems = articleContents.map((result) =>
      ArticleContentError.isError(result)
        ? new vscode.TreeItem("記事の取得に失敗しました")
        : new ArticleTreeItem(result)
    );

    return treeItems;
  }
}
```

`getChildren()` と `getTreeItem()` はクラスを実装する上で必須のメソッドで、`getChildren()` は `vscode.TreeItem` のインスタンスを含む配列を返すと、その TreeItem を表示し、`getTreeItem()` は `getChildren()` が実行された後に実行されます。

ここでのポイントは、`getArticleContents()` 内で発生したエラーはわざと throw せずに、返り値として返すようにしています。こうすることで、`ArticlesTreeViewProvider` 側で TreeItem の表示を制御することができますし、後述しますが並列で処理しやすくなります。

ここまで実装できたら、次は `ArticleTreeItem` を実装していきましょうー 🥎

## treeview/articleTreeItem.ts を実装する

`ArticleTreeItem` は、TreeView で一覧表示される要素の表示内容を実装するためのクラスです。
今回は以下のように実装して記事情報を表示しませう 👇

```ts:./src/web/treeview/articleTreeItem.ts
import * as vscode from "vscode";
import { ArticleContent, getArticleTitle } from "../schemas/article"; // この後実装します

/** 記事を表示するTreeItem */
export class ArticleTreeItem extends vscode.TreeItem {
  constructor(content: ArticleContent) {
    super("", vscode.TreeItemCollapsibleState.None);

    // VSCode のデフォルトの挙動を有効にするのに必要
    this.resourceUri = content.uri;

    // 記事のタイトルを TreeItem に表示する
    this.label = getArticleTitle({
      emoji: content.value.emoji,
      title: content.value.title,
      filename: content.filename,
    });

    // TreeItem をクリックしたときに対応するファイルを開く
    this.command = {
      command: "vscode.open",
      title: "記事ファイルを開く",
      arguments: [content.uri],
    };

    // 記事の状態を表示する
    this.description = [
      content.value.published ? "公開" : "非公開",
      content.value.slug,
    ]
      .join("・");
  }
}
```

ここでのポイントは**死ぬほどあります**。ｴｪ… (;´д ｀)

まず TreeItem を実装するクラスには**なるべくロジックを含まないようにした方がいいです。**
理由は、現状(2022/12)だと TreeView を更新する際に TreeView 全体を更新する必要があるため、どんなに更新内容が小さくても全体を更新する必要があります。そのため、思った以上に TreeItem は生成＆破棄されやすいので、そこにロジックを落ち込むとマジで面倒なバグに遭遇します( n 敗 )。

次に `this.resourceUri` は設定しておくと VSCode が持っているデフォルトの機能が有効になるので、設定できるなら設定しておいた方が良いです( 例えば、ファイルアイコンなどを自動で表示してくれたりする )。

`this.command` は設定すると、TreeItem をクリックしたときに設定したコマンドを実行できます。今回は `vscode.open` コマンドを使って記事ファイルを開けるようにしていますが、他にも便利なコマンドがあるので[公式ドキュメント](https://code.visualstudio.com/api/references/commands)を見て実装するといい思います。※ 自前で実装したコマンドも使用できます。

`this.description` は、`this.label` の横に薄く表示されるテキストで、Zenn の拡張では記事のステータスなどを表示するのに使っています。実際の表示は以下のような感じ 👇

![](/images/create-zenn-vscode-extension/treeitem-description.png)
_`公開・markdown-guide` が description の部分_

ただし、`this.label` が長いと省略されて表示されるので、重要な情報を載せるのは控えた方がいいかもしれないです。

はい、以上で `ArticleTreeItem` の実装は完了です。
次は記事情報を取得する処理を実装していきましょうー 🌵

## schemas/article.ts を実装する

次は、`./src/web/schemas/article.ts` に記事情報の具体的な処理を実装していきますが、処理に `yaml` パッケージが必要なのでインストールしておきます 👇

```shell:yamlをインストール
$> yarn add yaml
```

また yaml を import するとビルド時にエラーが出てしまいますので、エラーが出ないように `./webpack.config.js` を少し修正します 👇

```diff js:./webpack.config.js
  /* -- 省略 -- */

  /** @type WebpackConfig */
  const webExtensionConfig = {
    /* -- 省略 --*/
    plugins: [
      new webpack.optimize.LimitChunkCountPlugin({
        maxChunks: 1, // disable chunks by default since web extensions must be a single bundle
      }),
      new webpack.ProvidePlugin({
-       process: "process/browser", // provide a shim for the global `process` variable
+       process: "process/browser.js", // provide a shim for the global `process` variable
      }),
    ],
    /* -- 省略 -- */
  };

  module.exports = [webExtensionConfig];
```

これによって、yaml を import できるようになりましたので、さっそく処理を実装していきましょうー 🧅

### 型を実装する

まずは記事情報の型を以下のように実装しましょう 👇

```ts:./src/web/schemas/article.ts
import * as vscode from "vscode";

/** 記事のFrontMatterの情報 */
export interface Article {
  slug?: string;
  type?: "tech" | "idea";
  title?: string;
  emoji?: string;
  topics?: string[];
  published?: boolean;
  published_at?: string;
  publication_name?: string;
}

/** 記事の情報を含んだ型 */
export interface ArticleContent {
  value: Article;
  uri: vscode.Uri;
  filename: string;
  markdown: string;
}

/** 記事のエラーを扱うクラス */
export class ArticleContentError extends Error {
  static isError(value: unknown): value is ArticleContentError {
    return value instanceof ArticleContentError;
  }
}

/** 取得処理の結果型 */
export type ArticleContentLoadResult = ArticleContent | ArticleContentError;
```

`ArticleContentError` は TreeView で TreeItem の出し分けに使うので、あえて Error クラス使わずに継承したクラスにしています。

### `getArticleTitle()` を実装する

型定義ができたら、次は `ArticleTreeItem` 内で使用している `getArticleTitle()` を実装します 👇

```ts:./src/web/schemas/article.ts
import * as vscode from "vscode";

/* -- 省略 -- */

/** 記事のタイトルを返す */
export function getArticleTitle({
  emoji,
  title,
  filename,
}: {
  emoji?: string;
  title?: string;
  filename?: string;
}): string {
  if (title) return `${emoji}${title}`;
  if (filename) return `${emoji}${filename}`;

  return "タイトルが設定されていません";
}
```

本来であれば Emoji の判定などをした方がいいですが、今回は簡略化のため実装していません。もし実装したいなら [emoji-regex](https://github.com/mathiasbynens/emoji-regex) などを使うといいと思います。

### `getArticleContents()`を実装する

次に、記事一覧を取得する関数 `getArticleContents()` を実装していきます 👇

```diff ts:./src/web/schemas/article.ts
  import * as vscode from "vscode";
+ import { AppContext } from "../extension";

  /* -- 省略 -- */

  /** 記事のタイトルを返す */
  export function getArticleTitle() {/* --  省略 -- */}

+ /** 記事の一覧を返す */
+ export async function getArticleContents(
+   context: AppContext
+ ): Promise<ArticleContentLoadResult[]> {
+   const rootUri = context.articlesFolderUri;
+
+   // `./articles` 内のファイル一覧を取得
+   const files = await vscode.workspace.fs.readDirectory(rootUri);
+
+   // markdown ファイルのみの vscode.Uri 配列を返す
+   const markdowns = files.flatMap((file) =>
+     file[1] === vscode.FileType.File && file[0].endsWith(".md")
+       ? [vscode.Uri.joinPath(rootUri, file[0])]
+       : []
+   );
+
+   // 取得結果の配列を返す
+   return Promise.all(markdowns.map((uri) => loadArticleContent(uri)));
+ }
```

context に含まれる `articlesFolderUri` を使って取得する対象の Uri を取得し、この後実装する`loadArticleContent()` に渡しています。また、[Promise.all](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) を使うことで並列で取得処理を実行できるので、使うようにしています。

:::message
[Promise.all](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) は、一つでも reject された Promise があると、他の Promise を無視して reject してしまうので注意が必要です。`loadArticleContent()` では Error を throw しないで Error インスタンスを返すようにしています。
:::

### `loadArticleContent()`を実装する

`getArticleContents()`が実装できたら、次は渡された vscode.Uri から記事情報を返す関数 `loadArticleContent()` を実装します 👇

```diff ts:./src/web/schemas/article.ts
  import * as vscode from "vscode";
  import { AppContext } from "../extension";

  /* -- 省略 -- */

  /** 記事のタイトルを返す */
  export function getArticleTitle() {/* --  省略 -- */}

+ /** 記事情報を取得する */
+ export async function loadArticleContent(
+   uri: vscode.Uri
+ ): Promise<ArticleContentLoadResult> {
+   try {
+     return vscode.workspace
+       .openTextDocument(uri)
+       .then((doc) => createArticleContent(uri, doc.getText()));
+   } catch {
+     return new ArticleContentError("記事の取得に失敗しました");
+   }
+ }

  /** 記事の一覧を返す */
  export async function getArticleContents(){/* -- 省略 -- */}
```

ここの注意点は `vscode.workspace.openTextDocument()` が返す値は Promise ではなく Thenable なので、`.catch()` がありません。そのため [try...catch](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/try...catch) を使用して実装する必要があります。

## `createArticleContent()` を実装する

`loadArticleContent()` の実装ができたら、次は渡された Markdown 文字列から記事情報を生成する関数 `createArticleContent()` を実装します 👇

```diff ts:./src/web/schemas/article.ts
  import * as vscode from "vscode";
  import { AppContext } from "../extension";

  /* -- 省略 -- */

  /** 記事のタイトルを返す */
  export function getArticleTitle() {/* --  省略 -- */}

+ /** Markdown文字列から記事データを作成する */
+ export  function createArticleContent(
+   uri: vscode.Uri,
+   text: string
+ ): ArticleContent {
+   const filename = uri.path.split("/").slice(-1)[0];
+
+   return {
+     uri,
+     filename,
+     value: {
+       slug: filename.replace(".md", ""),
+       ...parseFrontMatter(text),
+     },
+     markdown: text.replace(FRONT_MATTER_PATTERN, ""),
+   };
+ };

  /** 記事情報を取得する */
  export async function loadArticleContent(){/* -- 省略 -- */}

  /** 記事の一覧を返す */
  export async function getArticleContents(){/* -- 省略 -- */}
```

`parseFrontMatter()` と `FRONT_MATTER_PATTERN` は未実装なのでエラーが出ていると思いますが、この後実装するので大丈夫です。処理自体も単純なオブジェクトを生成するだけなので、特に問題ないと思います。

## `parseFrontMatter()` を実装する

`createArticleContent()` が実装できたら、次は FrontMatter 文字列をオブジェクトに変換する `parseFrontMatter()` を実装します 👇

```diff ts:./src/web/schemas/article.ts
  import * as vscode from "vscode";
  import { AppContext } from "../extension";
+ import { parse as parseYaml } from "yaml";

  /* -- 省略 -- */

  /** 記事のタイトルを返す */
  export function getArticleTitle() {/* --  省略 -- */}

+ /** front matterを取得するための正規表現 */
+ export const FRONT_MATTER_PATTERN = /^(-{3}(?:\n|\r\n)([\w\W]+?)(?:\n|\r\n)-{3})/;
+
+ /** Front Matterをオブジェクトに変換する */
+ export function parseFrontMatter(text: string): Record<string, string | undefined> {
+   const frontMatter = FRONT_MATTER_PATTERN.exec(text)?.[2];
+   const result = frontMatter ? parseYaml(frontMatter) : {};
+
+   if (typeof result !== "object") return {};
+   if (Array.isArray(result)) return {};
+
+   return result;
+ }

  /** Markdown文字列から記事データを作成する */
  export function createArticleContent(){/* -- 省略 -- */}

  /** 記事情報を取得する */
  export async function loadArticleContent(){/* -- 省略 -- */}

  /** 記事の一覧を返す */
  export async function getArticleContents(){/* -- 省略 -- */}
```

正規表現を解説するのは難しいので、そういうもんだと理解してください (´・ω・｀)
FrontMatter からオブジェクトへの変換には冒頭でインストールした yaml を使用しています。

## TreeView を表示してみる

はい！ここまで実装できたら、ようやく TreeView の完成です！
実際にデバッグやコマンドを実行して、拡張を実行してみて TreeView が表示されていれば成功です ✨

:::message
TreeView の表示には `./articles/*.md` が必要です。無ければファイルを作成してから拡張を実行してください！
:::

![](/images/create-zenn-vscode-extension/show-treeview.gif)
_実装して TreeView が表示されている様子_

# 🖥️ Markdown プレビュー機能を実装する

次は Markdown ファイルをプレビューする機能を実装していきます。

[zenn-cli](https://github.com/zenn-dev/zenn-editor) と同じプレビュー内容にするために [WebView API](https://code.visualstudio.com/api/extension-guides/webview) を使用して実装しますが、そのためには WebView で動作させるための JavaScript ファイルが必要です。

Zenn の VSCode 拡張では React を用いて実装していますので、この記事でも React で実装していきたいと思います 🍟

## 必要なパッケージをインストール

っと、その前に必要なパッケージをあらかじめインストールしておきます。

```shell:必要なパッケージをインストール
# ソースコード内で必要なパッケージをインストール
$> yarn add react react-dom zenn-content-css zenn-embed-elements zenn-markdown-html
$> yarn add -D @types/react @types/react-dom @types/vscode-webview

# ビルド時に必要なパッケージをインストール
$> yarn add -D style-loader css-loader sass sass-loader esbuild-loader

# Webpack 5 はコアモジュールを自動的でポリフィルしなくなったので手動でインストールする
$> yarn add -D crypto-browserify stream-browserify
```

インストールが完了しましたら、さっそく WebView を実装していきましょう 🥓

## WebView 用の React アプリを作成する

まずは始めに、WebView の実装を含んだ全体のフォルダー構造は以下のようになります 👇

```diff:プロジェクト全体のフォルダー構造
  .
  ├─ src
  │  ├─ web
+ |  └─ webview
+ │     ├─ src
+ │     |  ├─ index.scss
+ │     |  └─ index.tsx
+ │     └─ tsconfig.json
  └─ package.json
```

これを踏まえて、まずは `tsconfig.json` を以下のように実装しましょう 👇

```json:./src/webview/tsconfig.json
{
  "compilerOptions": {
    "module": "ESNext",
    "target": "ESNext",
    "lib": ["dom", "ES2019"],
    "types": ["vscode-webview"],
    "sourceMap": true,
    "strict": true,
    "jsx": "react-jsx",
    "moduleResolution": "Node",
    "esModuleInterop": true,
    "isolatedModules": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["**/*.ts", "**/*.tsx"]
}
```

types フィールドで指定している `"vscode-webview"` は、あらかじめインストールした `@types/vscode-webview` パッケージです。このパッケージを読み込んでおくことで、WebView 上で VSCode API を扱いやすくなります。

### webview/src/index.tsx を実装する

つづけて WebView の起点となる `webview/src/index.tsx` を以下のように実装します 👇

```tsx:./src/webview/src/index.tsx
import "zenn-content-css";
import "./index.scss";

import { useEffect, useState } from "react";
import { createRoot } from "react-dom/client";

// VSCode API を使用すためのオブジェクトを取得する
const vscode = acquireVsCodeApi();

const App = () => {
  const [html, setHtml] = useState("");

  useEffect(() => {
    // 埋め込み要素を表示できるようにする
    import("zenn-embed-elements");

    // 拡張側からのイベントを受け取る関数
    const onMessage = (event: MessageEvent) => {
      const msg = event.data as { html?: string };

      if (msg && typeof msg.html === "string") {
        setHtml(msg.html);
      }
    };

    window.addEventListener("message", onMessage);

    // html をもらうためにイベントを送信する
    vscode.postMessage({ type: "ready" });

    return () => {
      window.removeEventListener("message", onMessage);
    };
  }, []);

  return (
    <div className="zncContainer">
      <div className="znc" dangerouslySetInnerHTML={{ __html: html }} />
    </div>
  );
};

// VSCodeのWebViewのデフォルトスタイルを削除する
if (typeof window !== "undefined") {
  const defaultStyle = document.getElementById("_defaultStyles");
  defaultStyle?.remove();
}

const container = document.getElementById("root") as HTMLElement;
const root = createRoot(container);

root.render(<App />);
```

基本的な動作としては、

1. 上記の `<App />` が WebView で表示される
2. `<App />` がマウントされた後、拡張側に `"ready"` イベントを送信します
3. 拡張側が `"ready"` イベントを受け取ると HTML 文字列を WebView 側に送信します
4. 送信された HTML 文字列を `onMessage()` で受け取る
5. `setHtml()` でステートが更新されるとプレビューが表示される

という流れに流れになります。

注意点として、WebView 内で VSCode API を実行する際は、`acquireVsCodeApi()` から返されるオブジェクトを介して API を実行しますが、この `acquireVsCodeApi()` は複数回実行するとエラーが発生するので、なるべくトップレベルで実行するなどしてエラーを出さない工夫をする必要があります。

また、WebView には VSCode のデフォルトのスタイルが追加されますが、今回は `zenn-content-css` と競合してしまう部分があるので、プレビューを描画する前に削除しています。

### webview/src/index.scss を実装する

次に、少し見栄えを良くするために最低限のスタイルを `index.scss` に実装します 👇

```scss:./src/webview/src/index.scss
:root {
  /* Colors */
  --c-body: rgba(0, 0, 0, 0.82);
  /* Font */
  --font-base: -apple-system, "BlinkMacSystemFont", "Hiragino Kaku Gothic ProN",
    "Hiragino Sans", Meiryo, sans-serif, "Segoe UI Emoji";
  --font-code: "SFMono-Regular", Consolas, "Liberation Mono", Menlo, monospace,
    "Segoe UI Emoji";
}
html {
  background: white;
  box-sizing: border-box;
  font-size: 16px;
  -ms-text-size-adjust: 100%;
  -webkit-text-size-adjust: 100%;
  line-height: 1.5;
}
*, *:before, *:after {
  box-sizing: inherit;
}
body {
  padding: 0;
  margin: 0;
  color: var(--c-body);
  word-break: break-word;
  word-wrap: break-word;
  font-size: 16px;
  font-family: var(--font-base);
}
img {
  max-width: 100%;
}
p, blockquote, dl, dd, dt, section {
  margin: 0;
}
a {
  text-decoration: none;
  color: inherit;
}
h1, h2, h3, h4, h5, h6 {
  margin: 0;
  font-weight: 700;
  line-height: 1.4;
  outline: 0;
}
ul, ol {
  margin: 0;
  padding: 0;
  list-style: none;
}
li {
  margin: 0;
  padding: 0;
}
hr {
  border: none;
}
button {
  font-family: inherit;
  border: none;
  cursor: pointer;
  appearance: none;
  background: transparent;
  font-size: inherit;
  font-weight: inherit;
  font-family: inherit;
  transition: 0.25s;
  padding: 0;
  margin: 0;
  line-height: 1.3;
  color: var(--c-body);
}
code {
  font-family: var(--font-code);
}
.zncContainer {
  max-width: 760px;
  margin: 0 auto;
  padding: 3rem 1.8rem;
}
```

特に難しい所はないと思うので大丈夫かと思います( 鉄の意志 )。
次は実装した React のソースコードをビルドできるようにしていきましょうー 💈

## WebView 用の React アプリをビルドできるようにする

上記の React.js のソースコードをビルドできるように、`./webpack.config.js` を以下のように実装しましょう 👇

```diff js:./webpack.config.js
  /* -- 省略 -- */
  const path = require("path");
  const webpack = require("webpack");

  /** @type WebpackConfig */
  const webExtensionConfig = {
    /* -- 省略 -- */
    resolve: {
      /* -- 省略 -- */
      fallback: {
        // Webpack 5 no longer polyfills Node.js core modules automatically.
        // see https://webpack.js.org/configuration/resolve/#resolvefallback
        // for the list of Node.js core module polyfills.
        assert: require.resolve("assert"),
+       crypto: require.resolve("crypto-browserify"),
+       stream: require.resolve("stream-browserify"),
      }
    },
    module: {
      rules: [
        {
          test: /\.ts$/,
          exclude: /node_modules/,
-         use: [
-           {
-             loader: "ts-loader",
-           },
-         ],
+         loader: "esbuild-loader",
+         options: {
+           loader: "ts",
+           target: "esnext",
+         },
        },
        /* -- 省略 -- */
      ]
    },
    /* -- 省略 -- */
  }

+ const IS_DEV = process.env.NODE_ENV !== "production";
+
+  /**
+   * WebView で表示するファイルをビルドするための設定
+   * @type WebpackConfig
+   */
+  const webviewConfig = {
+    target: "web",
+    mode: IS_DEV ? "development" : "production",
+    devtool: IS_DEV ? "inline-source-map" : void 0,
+    entry: {
+      preview: "./src/webview/src/index.tsx",
+    },
+    output: {
+      path: path.join(__dirname, "./dist/webview"),
+      filename: "[name].js",
+      clean: true,
+    },
+    module: {
+      rules: [
+        {
+          test: /\.tsx?$/,
+          exclude: /node_modules/,
+          use: {
+            loader: "esbuild-loader",
+            options: {
+              loader: "tsx",
+              target: "esnext",
+              tsconfigRaw: require("./src/webview/tsconfig.json"),
+            },
+          },
+        },
+        {
+          test: /\.css$/,
+          use: ["style-loader", "css-loader"],
+        },
+        {
+          test: /\.scss$/,
+          use: ["style-loader", "css-loader", "sass-loader"],
+        },
+      ],
+    },
+    resolve: {
+      extensions: [".tsx", ".ts", ".jsx", ".js", ".json"],
+      fallback: {
+        crypto: require.resolve("crypto-browserify"),
+        stream: require.resolve("stream-browserify"),
+      },
+    },
+    plugins: [new webpack.ProvidePlugin({ React: "react" })],
+  };
+
+ module.exports = [webExtensionConfig, webviewConfig];
```

ここでの注意点としては、Webpack 5 では Node.js のコアモジュールを自動的にポリフィルしないため、`resolve.fallback` で使うモジュールを設定する必要があります。今回は `crypto`・`stream` を設定しています。

また、[JSX transform](https://ja.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html) を使用するために `new webpack.ProvidePlugin({ React: 'react' })]` を指定しています。

ここまで実装できましたら、ビルドコマンドを実行してみてちゃんとビルドできるか確認してみましょう 👇

```shell
$> yarn compile-web
```

エラーが出てなければ大丈夫です 👌
次は、上記の実装を WebView で表示できるようにしていきましょうー 🍺

## プレビューコマンドの設定を実装する

WebView の実装が完了しましたら、次はプレビューコマンドの設定を実装します。package.json を以下のように実装しましょう 👇

```diff json:./package.json
  {
    /* -- 省略 -- */
    "contributes": {
+     "menus": {
+       "view/item/context": [
+         {
+           "command": "zenn-vscode-extension-example.preview",
+           "when": "viewItem == article",
+           "group": "inline"
+         }
+       ]
+     },
      "commands": [
        {
          "command": "zenn-vscode-extension-example.new-article",
          "title": "Zenn: New Article"
        },
+       {
+         "command": "zenn-vscode-extension-example.preview",
+         "title": "Zenn: Preview Contents",
+         "icon": "$(window)"
+       }
      ],
      /* -- 省略 -- */
    },
    /* -- 省略 -- */
  }
```

記事の作成コマンドの時と同様に、プレビューコマンドも `contributes.commands` に登録します。

そして、`contributes.menus` プロパティで TreeView 内の要素にプレビューコマンドのアイコンを表示することができます。このとき [when](https://code.visualstudio.com/api/references/when-clause-contexts) プロパティを使用することで、表示する要素を絞り込むことができます。今回は `"viewItem == article"` とすることで `ArticleTreeItem` のみに表示するようにしています。

また、ここの `viewItem` は `ArticleTreeItem.contextValue` プロパティの値を参照しますので、`ArticleTreeItem` に `contextValue` プロパティを追加します。

```diff ts:./src/web/treeview/articleTreeItem.ts
  /* -- 省略 -- */

  /** 記事を表示するTreeItem */
  export class ArticleTreeItem extends vscode.TreeItem {
+    // preview コマンドのアイコンを表示できるようにする
+    contextValue = "article";

    /* -- 省略 -- */
  }
```

これで `ArticleTreeItem` にプレビューコマンドのアイコンが表示されるようになりますが、**現時点ではプレビューコマンドを実装してないのでエラーになります。** なので、次はプレビューコマンドを実装していきましょうー 🏕

## プレビューコマンドを実装する

まずは `./src/web/commands/preview.ts` ファイルを作成して、そこにプレビューコマンドを実装しましょう 👇

```ts:./src/web/commands/preview.ts
import * as vscode from "vscode";
import { AppContext } from "../extension";
import ZennMarkdownToHtml from "zenn-markdown-html";
import {
  ArticleContentError,
  getArticleTitle,
  loadArticleContent
} from "../schemas/article";

/**
 * プレビューコマンドの実装
 */
export const previewCommand = (context: AppContext) => {
  return async (treeItem?: vscode.TreeItem) => {
    if (!treeItem?.resourceUri) {
      return vscode.window.showErrorMessage("プレビューできませんでした");
    }

    const article = await loadArticleContent(treeItem.resourceUri);

    if (ArticleContentError.isError(article)) {
      return vscode.window.showErrorMessage("プレビューできませんでした");
    }

    const title = `${getArticleTitle({
      emoji: article.value.emoji,
      title: article.value.title,
      filename: article.filename,
    })} のプレビュー`;

    // WebView パネルを作成する
    const panel = vscode.window.createWebviewPanel(
      "zenn-vscode-extension-example",
      title,
      { preserveFocus: true, viewColumn: vscode.ViewColumn.Two, },
      { enableScripts: true /* postMessageで通信するのに必要 */ }
    );

    // WebView 側からのイベントを受け取る
    panel.webview.onDidReceiveMessage((event?: { type: "ready" }) => {
      if (!event) return;
      if (event.type !== "ready") return;

      // Markdown 文字列を HTML 文字列に変換する
      const html = ZennMarkdownToHtml(article.markdown);

      panel.webview.postMessage({ html });
    });

    // React のスクリプトを読み込むための Uri を取得する
    const { extension: { extensionUri } } = context;
    const appUri = vscode.Uri.joinPath(extensionUri, "dist/webview/preview.js");
    const webviewSrc = panel.webview.asWebviewUri(appUri);

    // WebViewで表示するHTMLを設定する
    panel.webview.html =
      `<!DOCTYPE html>` +
      `<html lang="ja">` +
      `  <head>` +
      `    <meta charset="utf-8">` +
      `    <meta name="viewport" content="width=device-width,initial-scale=1.0">` +
      `    <title>${title}</title>` +
      `    <!-- 埋め込み要素のイベントを処理するためのスクリプト -->` +
      `    <script src="https://embed.zenn.studio/js/listen-embed-event.js"></script>` +
      `  </head>` +
      `  <body>` +
      `    <div id="root"></div>` +
      `    <script defer src="${webviewSrc}"></script>` +
      `  </body>` +
      `</html>`;
  };
};
```

やっていることとしては、引数に渡された TreeItem ( ArticleTreeItem ) から記事を取得して、その記事のプレビューを WebView を用いて表示しています。

上記の React で実装したソースコードは `asWebviewUri()` を使って読み込める Uri に変換する必要があります。また、**読み込む先はビルド後のファイルである点に注意してください。**

## プレビューコマンドを登録する

次にプレビューコマンドを登録して実行できるようにしましょう 👇

```diff ts:./src/web/context/commands.ts
  import * as vscode from "vscode";
  import { AppContext } from "../extension";
  import { newArticleCommand } from "../commands/newArticle";
+ import { previewCommand } from "../commands/preview";

  export const initializeCommands = (context: AppContext): vscode.Disposable[] => {
    return [
      // 記事ファイルの作成コマンド
      vscode.commands.registerCommand(
        "zenn-vscode-extension-example.new-article",
        newArticleCommand(context)
      ),
+    // Markdownファイルのプレビューコマンド
+    vscode.commands.registerCommand(
+      "zenn-vscode-extension-example.preview",
+      previewCommand(context)
+    ),
    ];
  };
```

# ✨ 完成

はい！遂にこれで完成です！

以下の Gif 画像のように、TreeView の要素から WebView パネルが開いてプレビューが表示されていれば成功です！

![](/images/create-zenn-vscode-extension/preview-markdown.gif)
_記事をプレビューしている様子_

以上、ここまでお疲れさまでした 🙌

# 📋 あとがき

ここまで読んでくれてありがとうございます。

書いていて思ったんですが、記事でやるような内容じゃないですね、ﾎﾝﾄ...

本来はもっとちゃんとやりたかったんですが、記事の文量が今の 10↑↑10 倍くらいになりそうだったので、心が折れました 😇 もし次やるなら Zenn の Book でやろうかと思いますが、需要あるんですかね 🙄？

あと、この記事で作った拡張はまともに使えたもんじゃないですが、記事の元になっている Zenn の VSCode 拡張はまともに使える機能が実装されていますので、より詳しい実装が見たい方はそちらを見て頂ければと思います 👇

https://github.com/zenn-dev/zenn-vscode-extension

**P.S. OSS で開発しているので、みなさんからのコントリビューションお待ちしています 🐈**

これが誰かの参考になれば幸いです。
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。

それではまた 👋
