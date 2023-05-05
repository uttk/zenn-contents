---
title: '良い子の諸君！VSCode 拡張のサイドバーは WebView も表示できるぞ！'
emoji: 🌟
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [vscode]
published: true
---

## どゆこと？

VSCode 拡張では、サイドバーをカスタマイズできます（以下の画像の Ⓑ の部分）

![](/images/vscode-side-panel-with-webiew/vscode-sidebar.png)
*https://code.visualstudio.com/docs/getstarted/userinterface より引用*

通常サイドバーにはファイルをツリー表示する TreeView ぐらいしか表示できませんが、実は WebView も表示することができます。[^1]

[^1]: https://code.visualstudio.com/updates/v1_50#_webview-views

しかし、この機能についての公式ドキュメントがあんまり無くて、参考になる情報が[公式サンプル](https://github.com/microsoft/vscode-extension-samples/tree/main/webview-view-sample)ぐらいしか無いので、この場を借りてサイドバーに WebView を表示する方法を紹介しようかなと思います 🐲

## 基本的な実装方法

:::message
既に VSCode 拡張の雛形を作成していることを前提に解説します。雛形の作成については [公式ドキュメント](https://code.visualstudio.com/api/get-started/your-first-extension) を参照してください。
また、ソースコードのみを見たい方は [公式サンプル](https://github.com/microsoft/vscode-extension-samples/tree/main/webview-view-sample) にソースコードがありますので、そちらを参考にして頂ければと思います。
:::

まずは `./package.json` に、これから実装する WebView の設定を追加します 👇

```json:./package.json
{
  // ...
  "contributes": {
    "views": {
      // 以下の設定では Explorer の下に WebView を表示できるようにします
      "explorer": [
        {
          "type": "webview",
          "id": "example.webview",
          "name": "WebView Sidebar"
        }
      ]
    }
  },
  // ...
}
```

package.json に設定を追加できたら、次は `./src/extension.ts` に WebView を登録するための処理を実装します 👇

```ts:./src/extension.ts
import * as vscode from "vscode";

// これから実装するので、この時点ではまだファイルが無い事に注意！
import { WebViewProvider } from "./WebViewProvider";

export function activate(context: vscode.ExtensionContext) {
  // WebView を登録
  context.subscriptions.push(
    vscode.window.registerWebviewViewProvider(
      "example.webview", // package.json で設定した`id`と同じ値にして下さい
      new WebViewProvider(context.extensionUri)
    )
  );
}

// This method is called when your extension is deactivated
export function deactivate() {}
```

次に、`WebViewProvider` クラスを実装します。
以下のコードは WebView を表示するための最小構成で、サイドバーに `Hello World!` を文字を表示します 👇

```ts:./src/WebViewProvider.ts
import * as vscode from "vscode";

export class WebViewProvider implements vscode.WebviewViewProvider {
  constructor(private extensionUri: vscode.Uri) {}

  public resolveWebviewView(webviewView: vscode.WebviewView) {
    // WebViewで表示したいHTMLを設定します
    webviewView.webview.html = `
      <!DOCTYPE html>
      <html lang="ja">
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>WebView Example</title>
      </head>
      <body>
        <h1>Hello World!</h1>
      </body>
      </html>
    `;
  }
}
```

`resolveWebviewView()` は必須です。この関数に WebView のインスタンスが渡されるので、そのインスタンスに対して表示内容を設定してあげることで、サイドバーに任意の HTML を表示することができます。

はい、これで実装は終わりです。TreeView の時より簡単に実装できますが、これだけだとできることが少ないと思いますので、以降は発展した使い方を紹介します。

## スクリプトを使用する

ほとんど [WebView API](https://code.visualstudio.com/api/extension-guides/webview#scripts-and-message-passing) 同じです。引数から受け取った WebView のインスタンスに `enableScripts` オプションを `true` に設定してあげることで使用できるようになります 👇

```ts:./src/WebViewProvider.ts
import * as vscode from "vscode";

export class WebViewProvider implements vscode.WebviewViewProvider {
  constructor(private extensionUri: vscode.Uri) {}

  public resolveWebviewView(webviewView: vscode.WebviewView) {
    webviewView.webview.options = {
      enableScripts: true, // スクリプトを使えるようにする
    };

    webviewView.webview.html = `
      <!DOCTYPE html>
      <html lang="ja">
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>WebView Example</title>
      </head>
      <body>
        <div id="app" />

        <script>
          const app = document.getElementById("app");
          app.innerText = "Hello World!";
        </script>
      </body>
      </html>
    `;
  }
}
```

上記の `<script />` 内のコードが実行されていれば OK です！

もし、`Blocked script execution in ~` みたいなエラーがコンソールに出ている場合は、`enableScripts` が正しく設定されていないので、ちゃんと `true` を設定してください。

## ローカルの JS ファイルを読み込む

プロジェクト内のローカルの JS ファイルを読み込めるようにするには以下のようにします 👇

```ts:./src/WebViewProvider.ts
import * as vscode from "vscode";

export class WebViewProvider implements vscode.WebviewViewProvider {
  constructor(private extensionUri: vscode.Uri) {}

  public resolveWebviewView(webviewView: vscode.WebviewView) {
    webviewView.webview.options = {
      enableScripts: true, // スクリプトを使えるようにする
    };

    // WebView 内で`./public/index.js`を読み込み可能にするためのUri
    const scriptUri = webviewView.webview.asWebviewUri(
      vscode.Uri.joinPath(this.extensionUri, "public", "index.js")
    );

    webviewView.webview.html = `
      <!DOCTYPE html>
      <html lang="ja">
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>WebView Example</title>
      </head>
      <body>
        <!-- ローカルのJSファイルを読み込んで実行します -->
        <script src="${scriptUri}" />
      </body>
      </html>
    `;
  }
}
```

上記の実装では、`./public/index.js` を WebView 内で読み込んで実行しています。
WebView 内でローカルファイルを読み込むためには `asWebviewUri()` を使って読み込み可能な Uri を取得する必要がある事に注意してください。

また、WebView はデフォルトで拡張内の全てのファイルを読み込めるようになっていますが、制限したい場合は `localResourceRoots` に適切な値を設定してください 👇

```diff ts:WebView内で読み込めるファイルを制限する
 webviewView.webview.options = {
   enableScripts: true,
+  // `./public` 内のファイルのみ読み込めるように制限する
+  localResourceRoots: [vscode.Uri.joinPath(this.extensionUri, 'public')]
 };
```

## ローカルの CSS ファイルを読み込む

JS ファイルの時と同様に `asWebviewUri()` を使って読み込み可能な Uri を生成して HTML に`<link />`を設定します。

```ts:./src/WebViewProvider.ts
import * as vscode from "vscode";

export class WebViewProvider implements vscode.WebviewViewProvider {
  constructor(private extensionUri: vscode.Uri) {}

  public resolveWebviewView(webviewView: vscode.WebviewView) {
    // 特に設定するべきオプションは必要ありませんが、
    // オブジェクトは代入する必要があります
    webviewView.webview.options = {};

    // WebView 内で`./public/index.css`を読み込み可能にするためのUri
    const styleUri = webviewView.webview.asWebviewUri(
      vscode.Uri.joinPath(this.extensionUri, "public", "index.css")
    );

    webviewView.webview.html = `
      <!DOCTYPE html>
      <html lang="ja">
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>WebView Example</title>

        <!-- ローカルのCSSファイルを読み込みます -->
        <link rel="stylesheet" href="${styleUri}">
      </head>
      <body>
        <h1 class="title">Hello World!</h1>
      </body>
      </html>
    `;
  }
}
```

## WebView と通信する(メッセージパッシング)

サイドバーの WebView でも拡張内のスクリプトと通信できます 👇

```ts:./src/WebViewProvider.ts
import * as vscode from "vscode";

export class WebViewProvider implements vscode.WebviewViewProvider {
  constructor(private extensionUri: vscode.Uri) {}

  public resolveWebviewView(webviewView: vscode.WebviewView) {
    // WebView にイベントを送信します
    webviewView.webview.postMessage({ type: "setup" });

    // WebView 内からのイベントを受け取ります
    webviewView.webview.onDidReceiveMessage((data) => {
      switch(data.type){
        case "any-event":
          console.log(data.text)
          break;
      }
    });

    webviewView.webview.html = `...`;
  }
}
```

WebView 内のスクリプトでは以下のようにします 👇

```ts:WebView内で実行されるスクリプト
// 拡張側からのイベントを受け取ります
window.addEventListener("message", (event) => {
  if(event.data.type === "setup") {
    console.log("setup now!")
  }
});

// acquireVsCodeApi() は一回だけ実行できます。
// 二回目以降はエラーが発生します。
const vscode = acquireVsCodeApi();

// 拡張側にイベントを送信します
vscode.postMessage({ type: "any-event", text: "Hello World!" })
```

おそらく上記のコードを見ただけだと、処理の流れが分かりにくいと思いますので、実際に実行してみて挙動を確認することをオススメします。

また、WebView が表示される前にイベントを送信してしまうと WebView 内で正しくイベントが受け取れないなどの問題があるため、イベントの送受信の処理タイミングはちゃんと把握しておくといいと思います。

## 永続化した値を取得する

WebView では任意の値を永続化できます。

https://code.visualstudio.com/api/extension-guides/webview#persistence

この永続化した値を取得するには、`resolveWebviewView()` の第二引数から取得します 👇

```ts:./src/WebViewProvider.ts
import * as vscode from "vscode";

export class WebViewProvider implements vscode.WebviewViewProvider {
  constructor(private extensionUri: vscode.Uri) {}

  public resolveWebviewView(
    webviewView: vscode.WebviewView,
    context: vscode.WebviewViewResolveContext,
  ) {
    // 永続化した値を取得する
    const state = context.state;

    webviewView.webview.html = `...`;
  }
}
```

## あとがき

以上で解説は終わりです。実装は [WebView API](https://code.visualstudio.com/api/extension-guides/webview) とほとんど同じで実装自体は難しくないんですが、自分で調べた感じ情報が全然無かったので、機能紹介みたいな感じで記事にしました。

サイドバーに WebView を表示できるようになると色々と面白い機能などが作れると思いますので、みなさんもこの機会に VSCode 拡張に入門してみてはいかがでしょうか？

これが誰かの参考になれば幸いです。
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。

それではまた 👋
