---
title: "葉桜の季節に君へ React 用ルーティングライブラリ wouter を紹介するということ"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, preact, wouter]
published: false
---

## この記事について

みなさん、こんにちは。
最近覚えた言葉は「ペトリコール」、uttk です。

ここ数日、Electron + React の環境を触っていて、その過程で久しぶりに自前でルーティング処理を実装することになり、色々とライブラリを探していると wouter というライブラリがシンプルで使いやすかったので、今回は wouter について少し解説したいと思います。

:::message
この記事は上記のリポジトリの README の情報を参考に書いています。
そのため、記事の情報やコードなどには引用などが含まれますので予めご了承ください 🙏
また、この記事の内容は時間とともに古くなる可能性があります。ライブラリの正しい使い方などについては、公式ドキュメントの方をなるべく参照して頂ければと思います 🙇‍♂️🙇‍♀️
:::

## wouter とは？

![](/images/introduction-of-wouter/wouter-icon.png)
_wouter の公式リポジトリより引用_

wouter は、[React](https://ja.react.dev/) または [Preact](https://preactjs.com/) で使用できるルーティングライブラリです。

https://github.com/molefrog/wouter

特徴としては以下のようなモノがあります 👇

- 依存関係がゼロ
- パッケージサイズが 4.3KB ( gzip の場合は 2KB )
- [React](https://ja.react.dev/) と [Preact](https://preactjs.com/) の両方をサポート
- `<Router />` がオプション
- SSR もサポートしている
- Tree Shaking しやすいパッケージ構造
- [React Router](https://reactrouter.com/en/main) のベストプラクティスが使える
- ルーティング (アニメーションなど) を制御するための Hooks API があります

一言でいうと「 軽量な React Router 」みたいな感じでしょうか。

2023/05/12 現在で最新のバージョンである `react-router@6.11.1` のパッケージサイズが 52KB[^1] なので、4.3KB はとても軽量だと思いますし、個人的に使っていてとてもシンプルに感じたので、特殊な要件などが無い場合は wouter で十分かなと思います。

[^1]: https://bundlephobia.com/package/react-router@6.11.1

さて、お次はそんな魅力が詰まった wouter の使い方について見ていきましょうー 🐲

## 基本的な使い方

まずは基本的な所から見ていきましょうー 🔫

```tsx:リンクとそれに対応したページを表示する例
import { Link, Route } from "wouter";

const ContactPage = () => {
  return <h1>Inbox Page</h1>;
};

const App = () => (
  <>
    <Link href="/about">About</Link>
    <Link href="/inbox" className="link">
      Inbox Page
    </Link>
    <Link href="/users/hoge">
      <a className="link">Profile</a>
    </Link>

    <Route path="/about">About Page</Route>
    <Route path="/contact" component={ContactPage} />
    <Route path="/users/:name">
      {(params) => <div>Hello, {params.name}!</div>}
    </Route>
  </>
);
```

上記の例では、リンクとそれに対応したページの表示方法をそれぞれ実装しています。

基本的な使い方は React Router を始めとするルーティングライブラリとほとんど変わりませんが、`<Router />` などのコンポーネントで `<Route />` を囲む必要がない点が特徴的です。

### `<Link />` について

上記の例では、`<Link />` をいくつかの方法で実装しています。
基本的にこのコンポーネントは `<a />` と同じように使えますが、children に `<a />` を使用する事もできます 👇

```tsx:上記の例より抜粋
<Link href="/users/hoge">
  <a className="link">Profile</a>
</Link>

// 上記の JSX は、以下のように描画されます
<a href="/users/hoge" className="link" />
```

注意点として `<a />` を children として使用する場合は、`className` は `<a />` に指定するようにして下さい。そうしないと、`className` が無視されます。

また、`<a />` 以外を表示するコンポーネントを children として使用することもできます 👇

```tsx
import { Link } from "wouter";

interface LinkButtonProps {
  href?: string; // <Link /> によって自動的に渡されます
  onClick?: () => void; // <Link /> によって自動的に渡されます
}

const LinkButton = (props: LinkButtonProps) => {
  return (
    <div title={props.href}>
      <button onClick={props.onClick}>Home</button>
    </div>
  );
};

const App = () => {
  return (
    <Link href="/home">
      <MyButton />
    </Link>;
  )
}
```

上記の例では、`<button />` をリンクとして動作させたいので、`<Link />` から渡される `onClick` を `<button />` のイベントに設定することでリンクとして動作させています。

ここでも注意点があり、[next/link](https://nextjs.org/docs/pages/api-reference/components/link) などでは [React.forwardRef()](https://ja.legacy.reactjs.org/docs/forwarding-refs.html) を使用する必要があります[^2]が、wouter では使う必要はありません。しかし、一応使うことはできるみたいなので、どうしても必要な場合は以下の issue を参考に実装してみるといいと思います 👇

https://github.com/molefrog/wouter/issues/287

[^2]: https://nextjs.org/docs/pages/api-reference/components/link#if-the-child-is-a-functional-component

### `<Route />` について

次にパスに対応した描画内容を表示する `<Route />` について見ていきたいと思います。ここでもいくつかの実装例がありますが、基本的には 3 通りの方法があります 👇

```tsx:<Route />の実装方法
import { Route } from "wouter";

// children に描画内容を指定する方法 (パラメーターは受け取れません)
<Route path="/">
  <Home />
</Route>

// children に関数を指定する方法 (パラメーターを受け取れる)
<Route path="/users/:id">
  {params => <UserPage id={params.id} />}
</Route>

// component を指定する方法 (props からパラメーターを受け取れる)
<Route path="/articles/:slug" component={ArticlePage} />
```

それぞれ使い方に大きな差はありませんが、パラメーターを受け取る場合は children に関数を指定してあげるか、component を渡してあげる必要があります。( P.S. パラメーターに関しては `<Route />` の場合だと型推論が弱いため、使用できるなら型が効きやすい [useRoute](#useroute) の方を使用することをオススメします )

また `path` のマッチングには React Router などで使用されている [path-to-regexp](https://github.com/pillarjs/path-to-regexp) を限定的に使っているため、

- 名前付き動的セグメント: `/users/:foo`
- 修飾子を含む動的セグメント: `/foo/:bar*`, `/foo/baz?`, `/foo/bar+`

の二つの方法でしか指定できません。もし他のマッチングを使用したい場合は [`path-to-regexp` の matcher を拡張する](#path-to-regexp-の-matcher-を拡張する) を参考にして下さい。

## 最初にマッチした Route だけを表示する

いわゆる「排他的なルーティング」と呼ばれる機能で、通常 `<Route />` だけを使うと `path` にマッチした全ての `<Route />` が表示されてしまいますが、動的なパスなどを使用している場合は最初にマッチした `<Route />` だけを表示したい時があります。

そのような場合は、`<Switch />` を使用することで想定する挙動を実現できます 👇

```tsx
<Switch>
  <Route path="/" component={TopPage} />
  <Route path="/users/all" component={AllUsersPage} />
  <Route path="/users/:id" component={UserPage} />

  {/* `path`を指定していない場合は全てのパスにマッチします */}
  <Route>Not Found 404</Route>
</Switch>
```

上記の例では、`/users/all` の場合は `<AllUsersPage />` のみを表示し、`/users/1` のような場合は `<UserPage />` を表示し、それ以外のパスは `Not Found 404` を表示するようにしています。

ここでの注意点は、**上から順番に `path` のマッチング処理が行われるため `/users/all` などのような固定化されたパスは上に、`/users/:id` のような動的なパスは下に配置する必要があります。**

これはつまり、以下のように `/users/all` と `/users/:id` の順番を逆にすると `/users/all` が一生表示されなくなります 👇

```tsx:"/users/all"が一生表示されなくなる例
<Switch>
  <Route path="/users/:id" component={UserPage} />

  {/* "/users/:id" の方が先にマッチするため、一生表示されません */}
  <Route path="/users/all" component={AllUsersPage} />
</Switch>
```

なので、`<Switch />` を使用する際は `<Route />` の順番に十分に注意しましょう 👩‍🏫

## Hooks API について

wouter にも便利な Hooks API がありますので、簡単ですが解説していきたいと思います 🥞

### useRoute

この Hooks を使用すると、パラメーターの取得などが簡単に行えるようになります 👇

```tsx:公式READMEより引用
import { useRoute } from "wouter";
import { Transition } from "react-transition-group";

const AnimatedRoute = () => {
  // `match` is boolean
  const [match, params] = useRoute("/users/:id");

  return <Transition in={match}>Hi, this is: {params.id}</Transition>;
};
```

この Hooks の良いところは、型推論によって `params` の型がちゃんとついている点です 👇

```ts:paramsの型
const [match, params] = useRoute("/users/:id");
match; // boolean
params; // { readonly id: string; } | null

const [match, params] = useRoute("/users/:id/:status");
match; // boolean
params; /*
  {
    readonly id: string;
    readonly status: string | undefined;
  } | null
*/
```

パラメーターを取得するのは `<Route />` でもできますが、`useRoute()` の方が型が付くので、こちらが使用できるならこちらの方を使用するといいかもしれません。

### useLocation

低レベルなナビゲーション処理を行いたい場合に、この Hooks を使用します 👇

```tsx:公式READMEより引用
import { useLocation } from "wouter";

const CurrentLocation = () => {
  const [location, setLocation] = useLocation();

  return (
    <div>
      {`The current page is: ${location}`}
      <a onClick={() => setLocation("/somewhere")}>Click to update</a>
    </div>
  );
};
```

上記の `location` は、現在のパス文字列です。( 例: `"/"`, `"/user"` )
また、`setLocation()` は、実行することで別のページへ遷移することができ、第二引数に `replace: true` を渡してあげることでリダイレクトと同じ処理を行うことができます 👇

```tsx:公式READMEより引用
const [location, navigate] = useLocation();

navigate("/jobs"); // `pushState` is used
navigate("/home", { replace: true }); // `replaceState` is used
```

注意点として、Hash ベースのルーティングなどの高度なナビゲーションを行いたい場合は `<Router />` を使用して独自の Hooks を使用する必要があります。

具体的な実装例は [ハッシュベースのルーティングを実装する](#ハッシュベースのルーティングを実装する) を参照してください。

### useRouter

この Hooks を使うと、直近の `<Router />` で定義された router オブジェクトを取得することができます 👇

```tsx:公式READMEより引用
import { useRouter } from "wouter";
import useLocation from "wouter/use-location";

const Custom = () => {
  const router = useRouter();

  // router.hook is useLocation by default

  // you can also use router as a mediator object
  // and store arbitrary data on it:
  router.lastTransition = { path: "..." };
};
```

また、取得できる Router の中身は以下のようになっています 👇

```ts:routerのプロパティ値
const router = useRouter()

router.base; // <Router base="..."> で設定した文字列。default: ""
router.ownBase; // base と同じ値。deafult: ""
router.parent; // 親のRouter。default: undefined
router.hook(); // <Router hook={...}> で設定した関数。default: useLocation
router.matcher(); // path のマッチング判定をする関数。default: path-to-regexp
```

`useRouter` は主に複雑なルーティング処理の実装に使う感じで、シンプルに使うなら使用頻度は高くないと思いますが、実装例として [ネストルーターの実装](#ネストルーターの実装) がありますので、詳しい使い方はそちらを参考にして下さい。

## Tips

公式 README に参考になる Tips がいくつか紹介されていましたので、この記事でも少し紹介しておきたいと思います 🍗

### `path-to-regexp` の matcher を拡張する

wouter では [path-to-regexp](https://github.com/pillarjs/path-to-regexp) を最小限にしか使っていないため、より高度なパスマッチングを行うには matcher 関数をカスタマイズする必要があります 👇

```tsx:公式READMEより引用
import { Router } from "wouter";

import makeCachedMatcher from "wouter/matcher";

/*
 * This function specifies how strings like /app/:users/:items* are
 * transformed into regular expressions.
 *
 * Note: it is just a wrapper around `pathToRegexp`, which uses a
 * slightly different convention of returning the array of keys.
 *
 * @param {string} path — a path like "/:foo/:bar"
 * @return {{ keys: [], regexp: RegExp }}
 */
const convertPathToRegexp = (path) => {
  let keys = [];

  // we use original pathToRegexp package here with keys
  const regexp = pathToRegexp(path, keys, { strict: true });
  return { keys, regexp };
};

const customMatcher = makeCachedMatcher(convertPathToRegexp);

function App() {
  return (
    <Router matcher={customMatcher}>
      {/* at the moment wouter doesn't support inline regexps, but path-to-regexp does! */}
      <Route path="/(resumes|cover-letters)/:id" component={Dashboard} />
    </Router>
  );
}
```

### ベースパスを指定する

`<Router />` を使うことで設定できます 👇

```tsx:公式READMEより引用
import { Router, Route, Link } from "wouter";

const App = () => (
  <Router base="/app">
    {/* the link's href attribute will be "/app/users" */}
    <Link href="/users">Users</Link>

    <Route path="/users">The current path is /app/users!</Route>
  </Router>
);
```

注意点として、独自の useLocation などを実装している場合は、useLocation に自前でベースパスの処理を実装する必要があります。

### デフォルトルートを作成する

全てのパスにマッチングする `<Route />` を実装するには以下のようにします 👇

```tsx:公式READMEより引用
import { Switch, Route } from "wouter";

<Switch>
  <Route path="/about">...</Route>
  <Route>404, Not Found!</Route>
</Switch>;
```

また、`<Switch />` 内の `<Route />` の順番は重要です。
動的な URL を設定している `<Route />` はなるべく下の方に配置するようにして下さい 👇

```tsx:公式READMEより引用
<Switch>
  <Route path="/users">...</Route>

  {/* will match anything that starts with /users/, e.g. /users/foo, /users/1/edit etc. */}
  <Route path="/users/:rest*">...</Route>

  {/* will match everything else */}
  <Route path="/:rest*">{(params) => `404, Sorry the page ${params.rest} does not exist!`}</Route>
</Switch>
```

### アクティブなリンクか判定する

`useRoute` に判定したいパス文字列を渡すことで判定することができます 👇

```tsx:公式READMEより引用
const [isActive] = useRoute(props.href);

return (
  <Link {...props}>
    <a className={isActive ? "active" : ""}>{props.children}</a>
  </Link>
);
```

### トレイリングスラッシュを含めてパスを指定する

[path-to-regexp](https://github.com/pillarjs/path-to-regexp) のオプションとカスタム matcher を使って実装することで対応可能です 👇

```tsx:公式READMEより引用
import makeMatcher from "wouter/matcher";
import { pathToRegexp } from "path-to-regexp";

const customMatcher = makeMatcher((path) => {
  let keys = [];
  const regexp = pathToRegexp(path, keys, { strict: true });
  return { keys, regexp };
});

const App = () => (
  <Router matcher={customMatcher}>
    <Route path="/foo">...</Route>
    <Route path="/foo/">...</Route>
  </Router>
);
```

### ネストルーターの実装

自前で実装する必要がありますが、簡単に実装することができます 👇

```tsx:公式READMEより引用
const NestedRoutes = (props) => {
  const router = useRouter();
  const [parentLocation] = useLocation();

  const nestedBase = `${router.base}${props.base}`;

  // don't render anything outside of the scope
  if (!parentLocation.startsWith(nestedBase)) return null;

  // we need key to make sure the router will remount when base changed
  return (
    <Router base={nestedBase} key={nestedBase}>
      {props.children}
    </Router>
  );
};

const App = () => (
  <Router base="/app">
    <NestedRoutes base="/dashboard">
      {/* the real url is /app/dashboard/users */}
      <Link to="/users" />
      <Route path="/users" />
    </NestedRoutes>
  </Router>
);
```

### パスを配列で指定したい

カスタム matcher を使うことで対応可能です 👇

```tsx:公式READMEより引用
import makeMatcher from "wouter/matcher";

const defaultMatcher = makeMatcher();

/*
 * A custom routing matcher function that supports multipath routes
 */
const multipathMatcher = (patterns, path) => {
  for (let pattern of [patterns].flat()) {
    const [match, params] = defaultMatcher(pattern, path);
    if (match) return [match, params];
  }

  return [false, null];
};

const App = () => (
  <Router matcher={multipathMatcher}>
    <Route path={["/app", "/home"]}>...</Route>
  </Router>
);
```

ただし、TypeScript を使うと型エラーが発生してしまうようなので、その点は注意です！
私の場合は、型定義を頑張っても恩恵が薄そうだったので `any` で妥協しました 😇

```tsx:型エラーを回避するためにanyを使用
const App = () => (
  <Router matcher={multipathMatcher as any}>
    <Route path={["/app", "/home"] as any}>...</Route>
  </Router>
);
```

### サーバーサイドレンダリング(SSR)サポート

アプリをトップレベルの `<Router />` でラップして、`ssrPath` を指定することで対応可能です 👇

```tsx:公式READMEより引用
import { renderToString } from "react-dom/server";
import { Router } from "wouter";

const handleRequest = (req, res) => {
  // top-level Router is mandatory in SSR mode
  const prerendered = renderToString(
    <Router ssrPath={req.path}>
      <App />
    </Router>
  );

  // respond with prerendered html
};
```

### ハッシュベースのルーティングを実装する

ハッシュを使ってルーティングするには `useLocation` をカスタマイズする必要があります 👇

```tsx:公式READMEより引用
import { useState, useEffect } from "react";
import { Router, Route } from "wouter";
import { useLocationProperty, navigate } from "wouter/use-location";

// returns the current hash location in a normalized form
// (excluding the leading '#' symbol)
const hashLocation = () => window.location.hash.replace(/^#/, "") || "/";

const hashNavigate = (to) => navigate("#" + to);

const useHashLocation = () => {
  const location = useLocationProperty(hashLocation);
  return [location, hashNavigate];
};

const App = () => (
  <Router hook={useHashLocation}>
    <Route path="/about" component={About} />
    ...
  </Router>
);
```

### リダイレクトについて

`useLocation` を使うやり方と、`<Redirect />` を使うやり方の 2 通りがあります 👇

```tsx:useLocationを使うやり方
import { useEffect } from "react";
import { useLocation } from "wouter";

const AnyComponent = () => {
  const [location, setLocation] = useLocation();

  useEffect(() => {
    fetchRequest("...").then(() => {
      setLocation("/redirect/to/page", { replace: true });
    });
  }, [])

  // ...
}
```

```tsx:<Redirect />を使うやり方
import { Redirect } from "wouter";

const AnyComponent = () => {
  const [isRedirect, redirect] = useState(false);
  const [location, setLocation] = useLocation();

  useEffect(() => {
    fetchRequest("...").then(() => redirect());
  }, [])

  if(isRedirect) return <Redirect replace to="/redirect/to/page" />

  // ...
}
```

どちらを使っても動作にそこまで違いはありませんので、状況によって使い分ければいいかと思います 🍠

### Preact で使う

wouter は [Preact](https://preactjs.com/) もサポートしているので、以下のように import 先を変えるだけで使えます 👇

```diff ts:公式READMEより引用
- import { useRoute, Route, Switch } from "wouter";
+ import { useRoute, Route, Switch } from "wouter-preact";
```

## あとがき

はい、以上で wouter の紹介は終了です。

シンプルかつ軽量なので、簡単な React アプリを作る時は選択肢に入れておくと良いかもしれません。だた、シンプル故に細かな動作については自前で実装する必要が出てくるので、その点は注意が必要ですね。( 公式 README に幾つか参考例があるので、ある程度はカバーできるかと思いますが )

これが誰かの参考になれば幸いです。
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。

それではまた 👋
