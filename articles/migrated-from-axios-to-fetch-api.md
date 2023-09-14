---
title: "Axios 使うのやめたらビルドサイズが 10 KB 減って、なんか知らんがパフォーマンスも良くなった話"
emoji: "⛪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "axios"]
published: true
publication_name: team_zenn
---

## この記事について

Zenn では長らく通信処理に Axios を使っていました。

https://github.com/axios/axios

しかし、[Fetch API](https://developer.mozilla.org/ja/docs/Web/API/Fetch_API) が多くのモダンブラウザなどで普通に使えるようになった今、使う必要性があまり無くなったため、Axios を使っている処理を全て [Fetch API](https://developer.mozilla.org/ja/docs/Web/API/Fetch_API) に置き換えることになりました。

この記事では、その置き換え作業をどう進めていったのか、その結果どう良くなったのかを解説していこうと思います 🗽

## 解説より置き換えた結果を知りたいのよ私は！！！

って方が居るかと思いますので、最初に置き換えたことで良くなった部分を紹介しようと思います。

:::message
結果はあくまで Zenn の環境下での話です。
Axios から Fetch API に移行したからといって、必ずしもパフォーマンスが向上するわけではありません。
:::


まず一番良くなったところといえば、ずばりサイト全体のビルドサイズが 10 KB も減りました。( ちなみに、10 KB は圧縮時のサイズで、圧縮しない場合 100 KB になります 😇 ﾜｰｵ )

![](/images/migrated-from-axios-to-fetch-api/bundle-analysis-ss.png)
*グローバルのビルドサイズが `103.35KB` gzip 時 10.07KB 減った*

これは今までのビルドサイズの 30 % になりますので、なかなかの削減に成功したことになります。

また、それに影響してか分かりませんが、フロントエンドサーバーのレイテンシが 40 % ほど良くなりました 👇

![フロントエンドのレイテンシが 0.13 秒から 0.08 秒に減った](/images/migrated-from-axios-to-fetch-api/paformance-graph-ss.png)
*フロントエンドのレイテンシが 0.13 秒から 0.08 秒に減った*

そもそも Zenn が速いのでパフォーマンスが良くなったとは言っても、人の感覚では分からないので、ユーザーへの影響はほとんどないですが、この結果はぜんぜん予期していなかったところだったので、個人的にはめっちゃ嬉しい結果でした！また、これにはチームメンバーもニッコリ 👽 ｱｸｼｵｽ ﾏｲ ﾌﾚﾝﾄﾞ

という感じで結果が分かった所で、お次は実際にどう置き換えていったのかについて解説していきます ⛵

## どのように置き換えていったか

Zenn の場合、以下のような順序で置き換え作業を行いました 👇

1. Fetch のラップ関数を実装する
2. 影響範囲の少なそうな所をラップ関数を使って置き換える
3. ラップ関数の動作確認( 👉 バグがあればこの時点で修正していく )
4. 動作が問題なければ、全体を置き換え ( 👉 この時にチェックリストも作っておく )
5. チェックリストを元に人力で動作確認する ( 👉 バグが見つかれば修正しておく )
6. 動作に問題が無ければ、本番に適用

本当は E2E テストや単体テストを活用して動作確認を楽にしたかったんですが、時間が無かったので今回は人力で頑張りました。協力してくれたチームメンバーには本当に感謝です 🙏

ほとんど参考になる部分は無いので申し訳ないんですが、`1. Fetch のラップ関数を実装する` 部分は参考になると思うので、次項で解説していきます 🏈

## Fetch のラップ関数を実装する

Zenn では Axios から Fetch API を置き換えるために、Fetch API をそのまま使うのではなく、Fetch API をラップした関数を使うようにしました。これには主に３つの理由があります。

1. Axios を使った既存のソースコードと似せて実装できる
2. ラップ関数の方が型を付けやすい
3. 共通処理を書きやすい

`1. Axios を使った既存のソースコードと似せて実装できる` の部分は置き換え作業の工程を減らすためにも絶対に必要なところです。ラップ関数を Axios と似せて実装することでコードの変更量をなるべく減らす努力をしています。

`2. ラップ関数の方が型を付けやすい` の部分は、Zenn が TypeScript を使っているためです。型を付けることによって、より TypeScript の恩恵をウケれるだけではなく、曖昧になっていた仕様が明確になるなど、色々な所で効果を発揮するため、今回の置き換え作業でも型を活用できるようにしています。

`3. 共通処理を書きやすい` の部分は、Zenn の場合だと主にスネークケース・キャメルケースの変更に対応するために必要でした。また、今後のことも考えると共通処理を入れやすい構造にした方がメリットが多いと思います。

と、前置きをしたところで、Fetch API をラップした関数を以下のように実装します 👇

```tsx:Fetch APIをラップした関数を実装
import camelcaseKeys from 'camelcase-keys';
import snakecaseKeys from 'snakecase-keys';

import { FetchError } from './custom-errors';

interface Options<T = object> {
  params?: T;
  headers?: HeadersInit;
  credentials?: Request['credentials'];
  validateStatus?: (status: number) => boolean;
}

/** 絶対URLかどうかを判定する　*/
function isAbsoluteURL(url: string): boolean {
  return /^([a-z][a-z\d+\-.]*:)?\/\//i.test(url);
}

/** URLとパスを連結する */
function combineUrls(baseURL: string, relativeURL: string): string {
  return relativeURL
    ? baseURL.replace(/\/+$/, '') + '/' + relativeURL.replace(/^\/+/, '')
    : baseURL;
}

/** URLを構築する */
function buildFullPath(baseURL: string, requestedURL: string): string {
  if (baseURL && !isAbsoluteURL(requestedURL)) {
    return combineUrls(baseURL, requestedURL);
  }
  return requestedURL;
}

/** リクエストヘッダを構築する */
function buildHeaders<T = HeadersInit>(headers?: T): HeadersInit {
  if (!headers) {
    // 未指定(undefined)の場合、`Content-Type: application/json` を返す
    return {
      'Content-Type': 'application/json',
    };
  }

  return headers;
}

/**
 * ローカル環境以外はセキュリティのためcredentialsをデフォルト値("same-origin")とする
 * @see https://developer.mozilla.org/ja/docs/Web/API/Request/credentials
 */
function buildCredentials(
  credentials?: Request['credentials']
): Request['credentials'] | undefined {
  if (process.env.NODE_ENV !== 'development') {
    return undefined;
  }

  return credentials;
}

/** リクエストボディを構築する */
function buildRequestBody<T = object>(body: T): string | FormData | null {
  // FormDataの場合、 `JSON.stringify()` せずそのまま返す
  if (body instanceof FormData) return body;

  // bodyがnull,undefinedの場合はnullを返して終了する
  // JSON.stringifyにnullを渡すとエラーになるため
  if (!body) return null;

  return JSON.stringify(
    snakecaseKeys(body as Parameters<typeof snakecaseKeys>)
  );
}

/** クエリパラメータ付きのURLパスを構築する */
function buildPathWithSearchParams<T = object>(path: string, params?: T) {
  // パラメータがない場合、URLパスをそのまま返す
  if (!params || Object.keys(params).length === 0) return path;

  for (const key in params) {
    if (params[key] === undefined) {
      // URLSearchParamsで`key="undefined"`になるので削除する
      delete params[key];
    }
  }

  const urlSearchParams = new URLSearchParams(params);
  return `${path}?${urlSearchParams.toString()}`;
}

/** 通信処理を共通化した関数 */
async function http<T>(path: string, config: RequestInit): Promise<T> {
  const request = new Request(
    // NEXT_PUBLIC_API_ROOTは必ず値が存在する想定なので `!` で型エラーを回避する
    buildFullPath(process.env.NEXT_PUBLIC_API_ROOT!, path),
    config
  );

  const res = await fetch(request);

  if (!res.ok) {
    const error = new FetchError('エラーが発生しました', {　status: res.status　});
    const data = await res.json();
    error.message = data.message;
    throw error;
  }

  // statusCodeが204のときにres.json()を実行するとエラーになるため
  if (res.status === 204) return {} as T;

  return camelcaseKeys(await res.json(), { deep: true });
}

export async function get<T, U = object>(
  path: string,
  options?: Options<U>
): Promise<T> {
  return http<T>(
    buildPathWithSearchParams(
      path,
      options?.params ? snakecaseKeys(options.params) : undefined
    ),
    {
      headers: buildHeaders(options?.headers),
      credentials: buildCredentials(options?.credentials),
    },
    options?.validateStatus
  );
}

export async function post<T, U, V = object>(
  path: string,
  body: T,
  options?: Options<V>
): Promise<U> {
  return http<U>(
    path,
    {
      method: 'POST',
      headers: buildHeaders(options?.headers),
      body: buildRequestBody(body),
      credentials: buildCredentials(options?.credentials),
    },
    options?.validateStatus
  );
}

export async function put<T, U = object>(
  path: string,
  body: T,
  options?: Options<U>
): Promise<U> {
  return http<U>(
    path,
    {
      method: 'PUT',
      body: buildRequestBody(body),
      headers: buildHeaders(options?.headers),
      credentials: buildCredentials(options?.credentials),
    },
    options?.validateStatus
  );
}

// deleteはJSの予約語であるためdestroyとする
export async function destroy<T = object>(
  path: string,
  options?: Options<T>
): Promise<unknown> {
  return http(
    buildPathWithSearchParams(
      path,
      options?.params ? snakecaseKeys(options.params) : undefined
    ),
    {
      method: 'DELETE',
      headers: buildHeaders(options?.headers),
      credentials: buildCredentials(options?.credentials),
    },
    options?.validateStatus
  );
}
```

上記の実装は主に以下の記事や [Axios](https://github.com/axios/axios) の実装を参考にしています 👇

https://eckertalex.dev/blog/typescript-fetch-wrapper

ポイントとしては、Axios と同じような感じで使えるように、`get`・`post`・`put`・`destroy(delete)` という関数を実装しています 👇

```ts
/** 通信処理を共通化した関数 */
async function http<T>(path: string, config: RequestInit): Promise<T> {
  /* ... */
}

export async function get<T, U = object>(/*...*/): Promise<T> {
  return http<T>(/* ... */);
}

export async function post<T, U, V = object>(/* ... */) {
  return http<T>(/* ... */);
}

export async function put<T, U = object>(/* ... */) {
  return http<T>(/* ... */);
}

export async function destroy<T = object>(/* ... */) {
  return http<T>(/* ... */);
}
```

:::message
`destroy()` の部分は元々 `delete()` でしたが、[delete](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/delete) が予約語のため、`destroy` としています。
:::

このようにする事で、Axios を使っている既存のコードに余り変更を加えなくて済むため、置き換えやすくなります 👇

```ts:実装を比べてみる
// before
import Axios from "axios";
Axios.get("/api/any", { withCredentials: true })

// After
import * as fetch from "@/utils/fetch";
fetch.get('/api/any', { credentials: 'include' })
```

また、TypeScript を使っているので、型引数を受け取れるようにしてなるべく型付けできるようにもしています 👇

```tsx:TypeScriptを使っているなら型を付けやすくしておくと便利
type Result = { message: string };
const result = await fetch.get<Result>("/api/any");
result.message; // string 型
```

これによって、普通に Fetch API を使うよりも型が効くようになって Type Guard のような処理が消せたりするので、より便利になったかなと思います。

他にも Zenn では、API に送るパラメータはスネークケース、フロントエンドで使う値は全てキャメルケースになっているため、それを考慮した処理も入れています 👇

```ts:リクエスト内容をスネークケースに変換する
/** リクエストボディを構築する */
function buildRequestBody<T = object>(body: T): string | FormData | null {
  /* ... */

  // リクエストする時はスネークケースでリクエストする
  return JSON.stringify(
    snakecaseKeys(body as Parameters<typeof snakecaseKeys>)
  );
}
```

```ts:APIレスポンス内容をキャメルケースに変換する
async function http<T>(path: string, config: RequestInit): Promise<T> {
  /* ... */

  // APIのレスポンスはスネークケースで返って来るのでキャメルケースに変換する
  return camelcaseKeys(await res.json(), { deep: true });
}
```

こういった変換処理は Axios だとミドルウェアで処理していたのですが、Fetch だとミドルウェアを用意するまでも無かったので、ミドルウェアなどの機構は実装せず、そのまま変換処理を挟む形で実装しています。

## あとがき

はい、という感じで解説は終わりです。

今回の Fetch API への移行は [catnose](https://zenn.dev/catnose99) さんが提案してくれたモノで、置き換え作業はとても大変でしたが、チームメンバーの助けもあって大きなバグもなく無事に Axios から Fetch API へ移行できたと思います。

結果としては、既に Zenn のパフォーマンスが良いので、使用感が劇的に変わった！みたいなことは無いんですが、「パフォーマンスを良くするために妥協しない！」という catnose さんの信念を感じられる非常に良い機会だったなと個人的に思っています。

記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。
これが誰かの参考になれば幸いです。

それではまた 👋