---
title: "そうです。わたしがReactをシンプルにするSWRです。"
emoji: "👴"
type: "tech"
topics: ["react", "typescript", "swr"]
published: true
---

# この記事について

[SWR](https://swr.vercel.app/)について色々と学んだので、その知見をここで共有したいと思います 💪

※ 基本的に以下の公式サイトの情報を参考にしています 📖

https://swr.vercel.app/

そのため、この記事で出すサンプルコードなどは主に上記の公式サイトから引用させて貰っています。予めご了承ください 🙏

# SWR とは何か？

[SWR](https://swr.vercel.app/)は、Next.js を作っている[Vercel 社](https://vercel.com/about)が開発しているデータフェッチのための[React Hooks](https://ja.reactjs.org/docs/hooks-intro.html)ライブラリです。"SWR"と言う名前は、`stale-while-revalidate`の頭文字をとって名付けられています。そのため、SWR は`stale-while-revalidate`に基づいた処理と設計になっています。

`stale-while-revalidate`について解説したい所ですが、解説するとすごく長くなってしまうため、ここでは「 **キャッシュをなるべく最新に保つ機能** 」という簡単なまとめ方で留めたいと思います。より知りたい方は RFC か以下のサイトが参考になります。

https://tools.ietf.org/html/rfc5861

https://blog.jxck.io/entries/2016-04-16/stale-while-revalidate.html

## SWR の特徴

- 非常にシンプル
- React Hooks ファースト
- 非同期処理を簡単に扱えるようになる
- 高速で軽量で再利用可能なデータフェッチ
- リクエストの重複排除
- リアクティブな動作の実現
- SSR / SSG に対応
- もちろん、Next.js 対応 🎊
- React Native でも使える 🎐

# 基本的な使い方

基本的な使い方を見て行きたいと思います 👀

```ts:公式サイトから引用
import useSWR from 'swr'

function Profile() {
  const { data, error } = useSWR('/api/user', fetcher)

  if (error) return <div>failed to load</div>
  if (!data) return <div>loading...</div>

  return <div>hello {data.name}!</div>
}
```

上記のソースコードでは、`useSWR`という[React Custom Hooks](https://ja.reactjs.org/docs/hooks-custom.html)をインポートしてコンポーネント内で使用していますが、少しコードが省略されている部分があるので補足します。

具体的には以下の部分の`fetcher`という所です。

```ts
const { data, error } = useSWR("/api/user", fetcher);
```

この`fetcher`は、名前から察せられるようにデータを取得する為の関数です。
例えば以下のような関数です。

```ts:fetcherの実装内容
const fetcher = (url: string): Promise<any> => fetch(url).then(res => res.json());
```

上記の`fetcher`では [fetch API](https://developer.mozilla.org/ja/docs/Web/API/Fetch_API/Using_Fetch) を使っていますが、Promise を返す関数であれば別に [fetch API](https://developer.mozilla.org/ja/docs/Web/API/Fetch_API/Using_Fetch) を使う必要はありません。Promise に対応させていれば好きなライブラリを使用することが可能です。

SWR の公式サイトでは、いくつかのライブラリの実装例を示してくれています。

https://swr.vercel.app/docs/data-fetching

次に、`useSWR()` が返す値について見てみます。

```ts
const { data, error } = useSWR("/api/user", fetcher);
```

`useSWR()` の返り値として、`data`・`error`をソースコードでは受け取っていますが、`data`には fetcher が resolve した値( 通信結果 )、もしくは`undefined`が入っています。

これは、

- `data` が `undefined` だと -> ロード中
- `data` が `Promise.resolve()` した値だと -> 通信終了

を意味しています。そのため、**`isLoading` なんてフラグは無いです 🐧**

`error`も同じような感じで、

- `error` が `undefined` だと -> エラーが発生していない
- `error` が `Promise.reject()` した値だと -> エラーが発生した

となります。

この値を元にコンポーネント内で分岐して表示を切り替えることができますので、サンプルコードでも対応した分岐が書かれています。

```ts:公式サイトから引用
function Profile() {
  const { data, error } = useSWR('/api/user', fetcher)

  if (error) return <div>failed to load</div>
  if (!data) return <div>loading...</div>

  return <div>hello {data.name}!</div>
}
```

## 条件付きフェッチ

次は、条件付きフェッチについて見て行きましょう。

条件付きフェッチとは文字通り値によって、フェッチするかしないかを判断して処理する方法です。ソースコードを見てみましょう。

```tsx:公式サイトより引用したものを少し改変
// 条件分岐でフェッチ
const { data } = useSWR(shouldFetch ? '/api/data' : null, fetcher)

// 関数を渡すこともできる
const { data } = useSWR(() => shouldFetch ? '/api/data' : null, fetcher)

// 第一引数の関数内でエラーを発生させても挙動としては同じようになる
// この時のエラーは、返り値の error で取得できないので注意！
const { data } = useSWR(() => '/api/data?uid=' + user.id, fetcher)
```

上記のソースコードから分かると思いますが `useSWR()` の第一引数に**falsy な値**を渡すと、fetcher の実行を停止してくれます。そのため、三項演算子などを使って引数の値を変更しています。

**falsy な値**については、以下の MDN のドキュメントから確認できますが、JavaScript( TypeScript )は**falsy な値**が結構多いです。なので、知らず知らずのうちに**falsy な値**を返してしまう事があるので、注意しましょう 📌

https://developer.mozilla.org/ja/docs/Glossary/Falsy

また、一番最後の例ですが

```ts:該当の例
// 第一引数の関数内でエラーを発生させても挙動としては同じようになる
// この時のエラーは、返り値の error で取得できないので注意！
const { data } = useSWR(() => '/api/data?uid=' + user.id, fetcher)
```

コメントにも書いてある通り、引数の関数がエラーを発生させた場合もリクエストの実行をしなくなります。**ただこちらは、使うべきではありません！**

何故なら、エラーが発生していても `useSWR()` はそのエラーを検出してくれません 😢
そのため、仮に第一引数の関数にバグが発生していても気づくのが難しくなってしまいます。なので、**できれば第一引数の値には関数は使わないようにした方が良いと思います。**

後、以下のように `if文` を使った分岐は[React Hooks の規約](https://ja.reactjs.org/docs/hooks-rules.html)に違反しているため、書くことができません。注意しましょう！

```ts:このような書き方はできません🙅‍♂️🙅‍♀️
if( shouldFetch ) {
  useSWR('/api/data', fetcher)
}
```

## 依存フェッチ

条件付きフェッチを応用して、複数のフェッチを連携させる(依存フェッチともいう)事が可能になります。サンプルコードを見てみましょう。

```ts:公式サイトより引用したものを少し改変
function MyProjects () {
  const { data: user } = useSWR('/api/user', fetcher)
  const { data: projects } = useSWR(() => '/api/projects?uid=' + user.id, fetcher)
  // 関数を渡す場合、SWRは返り値を`key`として使用します。
  // 関数がスローまたは falsy な値を返す場合、
  // SWRはいくつかの依存関係が準備できてないことを知ることができます。
  // この例では、`user.id`は`user`がロードされてない時にスローします。

  if (!projects) return 'loading...'

  return 'You have ' + projects.length + ' projects'
}
```

上記のソースコードでの、`/api/projects` へのリクエストは `'/api/user'` が完了してからしか実行されません。

また、何らかの要因で `'/api/user'` のリクエスト結果(この場合は`user`) が変更した場合は、自動的に `/api/projects` へのリクエストが発生します。これにより `projects` の値は、常に `user` に対してデータの整合性を保つようになっています。

条件付きフェッチの応用ですが、とても有用なテクニックなので、是非ともマスターしておきたいテクニックですね！

因みに、前述した通り`useSWR()`の第一引数には**関数を指定するべきではありません！**
なので、上記のソースコードは以下のようにした方が良いです 👇

```diff ts:上記のソースコードを修正
function MyProjects () {
  const { data: user } = useSWR('/api/user', fetcher)
- const { data: projects } = useSWR(() => '/api/projects?uid=' + user.id, fetcher)
+ const { data: projects } = useSWR(user ? `/api/projects?uid=${user.id}` : null, fetcher)
  // 関数を渡す場合、SWRは返り値を`key`として使用します。
  // 関数がスローまたは falsy な値を返す場合、
  // SWRはいくつかの依存関係が準備できてないことを知ることができます。
  // この例では、`user.id`は`user`がロードされてない時にスローします。

  if (!projects) return 'loading...'

  return 'You have ' + projects.length + ' projects'
}
```

## パラメータ付きのフェッチ

リクエストにパラメータを付けたい事が多いと思いますが、 `useSWR()` にはパラメータを文字列以外にも配列として渡すことができます。サンプルコードを見てみましょう。

```ts:公式サイトより引用
const { data: user } = useSWR(['/api/user', token], fetchWithToken)
```

上記のソースコードでは、`useSWR()` の第一引数に配列を渡しています。
これにより、 `useSWR()` に `token` を認識させることが可能となり、URL にパラメータを含まないようなリクエストにも対応します。

ここでの注意点として、 `useSWR()` は**配列の内容を浅い比較しかしません。** そのため、引数の値を渡す時には注意を払う必要があります。

```ts:よくない例(公式サイトより引用したものを少し改変)
// これはやらないで下さい！Depsはレンダリングごとに変更されるようになります！
// そのため通信結果が取得できなくなります！
useSWR(['/api/user', { id }], fetcher, query)

// 代わりに、「安定した」値のみを渡す必要があります。
useSWR(['/api/user', id], fetcher, (url, id) => query(url, { id }))
```

また、引数に渡した配列の値は fetcher の引数に渡されます。

```ts
const myFetcher = (url: string, id: number) => {
  console.log(`url ==> ${url}, id ==> ${id}`);
  // ...
};

useSWR(["/api/user", 1], myFetcher); // log出力結果: "url ==> /api/user, id ==> 1"
```

## fetcher は省略できる

:::message alert
SWR 1.0 以降から`<SWRConfig />`を使用した場合にのみ、fetcher を省略できるようになりました。詳しくは [こちら](https://swr.vercel.app/ja/blog/swr-v1#デフォルトのフェッチャーはもうありません) をご覧ください。
:::

後述する`<SWRConfig />`の`fetcher`オプションを使うことで、デフォルトの fetcher を設定できます。具体的には、以下のようにします 👇

```tsx:fetcher部分を省略する方法
import useSWR, { SWRConfig } from 'swr'

// <SWRConfig />を使って、デフォルトのfetcherを定義する
const App = () => {
  return (
    <SWRConfig value={{ fetcher: (url) => fetch(url).then(res => res.json()) }}>
      <UserList />
    </SWRConfig>
  )
}

const UserList = () => {
  const { data } = useSWR("/api/user"); // fetcherを省略できる！

  // ...
}
```

扱う際には注意が必要ですが、他のライブラリと一緒に使うことで、より簡潔に書けるようになるので、覚えておいて損はないですね！

これで基本的な使い方の紹介は終了です ✨

# グローバル設定( Global Configuration )

`useSWR()` の第三引数には、オプションを渡すことができます。

```ts:useSWRにオプションを渡す例
useSWR("/api/user", fetcher, options);
```

options の詳細は下の方に書いてありますので、[ぜひ参考にして下さい](#オプションについて)🙏

しかし、`useSWR()` にオプションは渡せても使うたびに渡さないといけないようでは、とても扱いづらくなってしまいます。そこで、`<SWRConfig />` を使う事で設定を共通化することができます。

```tsx:公式サイトより引用したものを少し改変
import useSWR, { SWRConfig } from 'swr'

function Dashboard () {
  const { data: events } = useSWR('/api/events')
  const { data: projects } = useSWR('/api/projects')
  const { data: user } = useSWR('/api/user', { refreshInterval: 0 }) // 設定を上書き
  // ...
}

function App () {
  // SWRConfig以下のコンポーネントにはグローバルな設定が反映される！
  return (
    <SWRConfig
      value={{
        refreshInterval: 3000,
        fetcher: (resource, init) => fetch(resource, init).then(res => res.json())
      }}
    >
      <Dashboard />
    </SWRConfig>
  )
}
```

:::message alert
ただし `useSWR()` にオプションが渡されている場合は、そちらのオプションの方が優先されるので注意が必要です。
:::

## グローバル設定の取得

`useSWRConfig()` を使うことで、グローバル設定を取得することができます。

```ts:グローバル設定を取得する例
import { useSWRConfig } from 'swr'

const Component = () => {
  // グローバル設定を取得する
  const { refreshInterval, mutate, cache, ...restConfig } = useSWRConfig();

  /* ... */
}
```

基本的には、後述する `mutate()` や Cache Provider を取得するために使用することが多いと思います。
また、ネストされた設定の場合は拡張(マージ)された設定値が返りますが、 `<SWRConfig />` を使っていない場合は、デフォルトの設定値を返します。

# Global Error

上記で解説した `<SWRConfig />` の `onError` オプションにコールバックを設定する事で、エラーをグローバルに処理することができます。

```tsx:公式サイトより引用
<SWRConfig value={{
  onError: (error, key) => {
    if (error.status !== 403 && error.status !== 404) {
      // We can send the error to Sentry,
      // or show a notification UI.
    }
  }
}}>
  <MyApp />
</SWRConfig>
```

アラート系のライブラリと組み合わせると、シンプルにエラー処理が実装できますので、どんどん活用していきましょう 🤘

# Error Retry

SWR では fetcher がエラーを発生した場合、fetcher を [exponential backoff アルゴリズム](https://en.wikipedia.org/wiki/Exponential_backoff) を使用して再実行します。しかし、場合によってはこれは必要ないかもしれません。なので、`onErrorRetry`オプションを使用して、この動作をオーバーライドすることが可能です。

```ts:公式サイトより引用
useSWR('/api/user', fetcher, {
  onErrorRetry: (error, key, config, revalidate, { retryCount }) => {
    // 404では再試行しない。
    if (error.status === 404) return
    // 特定のキーでは再試行しない。
    if (key === '/api/user') return
    // 再試行は10回までしかできません。
    if (retryCount >= 10) return
    // 5秒後に再試行します。
    setTimeout(() => revalidate({ retryCount: retryCount + 1 }), 5000)
  }
})
```

このコールバックにより任意のタイミングで fetcher を再試行できますが、そもそも再試行する必要が無い場合もあると思います。その時は、 `shouldRetryOnError: false` を指定する事により再試行を無効にすることが可能です。

# Mutation

SWR で、GET 以外の POST や DELETE といった処理には Mutation を使うことで対応できます。
Mutation を使う方法は、筆者が考えるに 4 通りありますので、この節では、その方法とその方法に対する筆者の知見を紹介したいと思います。

※ 方法の名前は分かりやすいさの為に、一部勝手に付けています。ご了承ください。

## 方法その１( Bound Mutate )

**Bound Mutate は、一番オススメの方法です。**
先ずは、ソースコードを見てみましょう。

```ts:BoundMutateのソースコード
const DisplayCatName = () => {
  const { data: cat, mutate } = useSWR("/cat", fetcher);

  const onUpdateCatName = async (catName: string) => {
    await postCatName(catName); // ネコの名前を更新する非同期関数

    mutate({ ...cat, name: catName }); // ここでSWRに変更を通知する
  };

  /* -- 省略 -- */
}
```

上記のソースコードでは、`useSWR()` が返すオブジェクトの中に `mutate` と言う関数があります。これに更新したい内容を渡すことで、**SWR にキャッシュの更新を通知することができます。**

この方法の良い所は、TypeScript を使っていれば `mutate` の引数に型が付いているので、**データの整合性が保つ事ができて扱いやすい事**と、**refetch(再取得)などの処理を抑えることができる事**です。

悪い所を上げるとするならば、`useSWR()` を実行する必要があるため、他のコンポーネントとの連携がやりにくいという欠点があります。しかし、それを補う機能が SWR にはありますので、そこまで気にする必要はありません！

## 方法その２( Refetch Mutation )

Refetch Mutation は、名前が示す通り refetch(再取得)によってデータの整合性を保つ方法です。ソースコードを見てみましょう。

```ts:RefetchMutationのソースコード
import useSWR, { useSWRConfig } from "swr";

const DisplayCatName = () => {
  const { mutate } = useSWRConfig();
  const { data: cat } = useSWR("/cat", fetcher);

  const onUpdateCatName = async (catName: string) => {
    await postCatName(catName); // ネコの名前を更新する非同期関数

    mutate("/cat"); // ここでSWRに変更を通知するが、文字列だけ渡している事に注意！

    // もしキーを配列で渡している場合は以下のようにします
    // mutate(["/cat"]);
  };

  /* -- 省略 -- */
}
```

Bound Mutate と違う所は、 `mutate` 関数を `useSWRConfig()` から取得していることと、実行時にキー文字列( 今回は "/cat" )だけを渡す必要があるという事です。

このように実行する事によって、SWR に refetch(再取得)を実行するように指示することができます。**これによって、サーバーとの間でデータの整合性を保つことができます！** サーバーとの同期が多く必要なサービスや、サーバー上で複雑な処理をしている場合は、重宝する方法ですね。

注意点を上げるとすると、関数の名前が Bound Mutate と同じ `mutate` なので名前の競合が起きやすい事と、**多用しすぎるとサーバーに負荷がかかりすぎる**ので、注意しましょう！👩‍🏫

:::message
ソースコード内のコメントにも書いてありますが、useSWR()の第一引数に配列を渡している場合は、mutate()でも同じ値を持った配列を渡す必要があります。
:::

## 方法その３( Update Local Mutate )

Update Local Mutate は、Refetch Mutation とほとんど同じような使い方です。
ソースコードを見てみましょう。

```ts:UpdateLocalMutateのソースコード
import useSWR, { useSWRConfig } from "swr";

interface Cat {
  id: number;
  name: string;
}

const DisplayCatName = () => {
  const { mutate } = useSWRConfig();
  const { data: cat } = useSWR<Cat>("/cat", fetcher);

  const onUpdateCatName = async (catName: string) => {
    await postCatName(catName); // ネコの名前を更新する非同期関数

    // ここでSWRに変更を通知する
    // 第三引数にfalseを渡すことで、再検証( 再取得 )する事を防ぐことができる
    mutate("/cat", { ...cat, name: catName }, false);

    // 因みにPromiseが更新内容を返すなら、そのPromiseをそのまま渡すこともできる
    // 仮に postCatName() の返り値が Promise<Cat> なら以下のように渡すことができる
    // mutate("/cat", postCatName(catName));
  };

  /* -- 省略 -- */
}
```

実行方法は Refetch Mutation とほとんど同じですが、`mutate()` の第二引数に更新する値を渡している所が違います。

このやり方が便利な所は、refetch(再取得)が発生しない事と、別の場所で使われている `useSWR()` に変更を通知することができます。これによって、離れた位置のコンポーネントの状態などを変更することができるので、**うまく使えばサーバーへの負荷を抑えつつキャッシュをより有効活用することができます 💪**

ただ注意としては、使いすぎると複雑なバグを引き起こしかねないので、Bound Mutate が使える場合は、そちらを使った方が良いと思います 🤔 ※ Boud Muatate だと範囲を限定できるので、バグを限定的にすることができます。

## 方法その４( Mutate Based on Current Data )

最後に Mutate Based on Current Data を紹介します。
ソースコードを見てみましょう。

```ts:MutateBasedOnCurrentDataのソースコード
import useSWR, { useSWRConfig } from "swr";

interface Cat {
  id: number;
  name: string;
}

const DisplayCatName = () => {
  const { mutate } = useSWRConfig();
  const onUpdateCatName = async (catName: string) => {
    mutate("/cat", async (cat: Cat): Promise<Cat> => {
      const newCat = { ...cat, name: catName };

      await postCat(newCat); // ネコの情報を更新する非同期関数

      return newCat; // ここで新しい値を返す必要がある！
    });
  };

  /* -- 省略 -- */
}
```

こちらも Refetch Mutation や Update Local Mutate と同じような使い方ですが、`mutate()` の第二引数に非同期関数( async function )を渡している点が違います。引数に渡した非同期関数は、 `useSWR()` が持っているキャッシュを受け取り、次のキャッシュの値を返します。

この方法は、現在取得しているデータを用いた変更処理を行う時に大変便利な方法です。これによって、わざわざ `useSWR()` を実行して値を取得する必要がありませんので、コンポーネントの描画を抑えることができますし、他の方法ではできない複雑な処理を実行する事も可能となっています。

ただ、こちらの方法はデータの整合性を保つのが難しい(引数で受け取る値が any)ですし、複雑なロジックが実行できるという事は、それだけバグを発生させやすいという事でもありますので、多用は禁物だと思います 🦉

## Mutation まとめ

上記の方法を筆者の知見を交えてまとめたいと思います。

| 方法名                       | メリット                                     | デメリット                             | オススメ度 |
| ---------------------------- | -------------------------------------------- | -------------------------------------- | ---------- |
| Bound Mutate                 | 簡単にデータを更新でき、範囲を限定的にできる | 複雑な処理はできない、 useSWR 必須     | 高         |
| Refetch Mutation             | データの整合性を高めることができる           | 非同期処理が走るため処理が遅くなるかも | 中         |
| Update Local Mutate          | useSWR を使わなくてもキャッシュを更新できる  | Bound Mutate で代用できることが多い    | 低         |
| Mutate Based on Current Data | 複雑な処理が可能                             | バグを引き起こしやすい                 | 中         |

# ページネーション( Pagination )

次はページネーション機能について見てきたいと思います。
先ずはサンプルコードを見てみましょう。

```ts:公式サイトより引用
function App () {
  const [pageIndex, setPageIndex] = useState(0);

  // The API URL includes the page index, which is a React state.
  const { data } = useSWR(`/api/data?page=${pageIndex}`, fetcher);

  // ... handle loading and error states

  return <div>
    {data.map(item => <div key={item.id}>{item.name}</div>)}
    <button onClick={() => setPageIndex(pageIndex - 1)}>Previous</button>
    <button onClick={() => setPageIndex(pageIndex + 1)}>Next</button>
  </div>
}
```

ページネーション機能とはいっても、 `useState()` でページのインデックスを管理し、それを `useSWR()` のパラメータとして使っているだけです。基本的な使い方の応用ですね。

`useSWR()` は、一度取得したデータをキャシュとして保持するので、一回取得してしまえば後は高速に動作します。これがとてもシンプルな実装でできるのはとても良いですよね ✨

しかし、無限ローディングなどの終わりのないページネーションや取得したデータ全体を扱いたい場合は、上記の方法では少し面倒な実装になってしまいます。そこで、SWR 側が `useSWRInfinite()` という Hooks を用意してくれています！やったね！

`useSWRInfinite()` を使う事で、簡単に無限ローディングなど実装することができます。

```ts:公式サイトより引用したものを少し改変
// 各ページのSWRキーを取得する関数
// 返り値は `fetcher` に渡されます
// `null` が返された場合、そのページのリクエストは開始されません。
const getKey = (pageIndex: number, previousPageData: any[]) => {
  if (previousPageData && !previousPageData.length) return null // 最後のページに到達した

  return `/users?page=${pageIndex}&limit=10` // fetcherの第一引数に渡される値

  // 配列を渡すこともできます。
  // 配列の場合は、fetcher(...[`/users`, pageIndex, 10])として展開されます
  // return [`/users`, pageIndex, 10]
}

function App () {
  const { data, size, setSize } = useSWRInfinite(getKey, fetcher)

  if (!data) return 'loading'

  // 取得した全データを使って計算できます
  let totalUsers = 0
  for (let i = 0; i < data.length; i++) {
    totalUsers += data[i].length
  }

  return (
    <div>
      <p>{totalUsers} users listed</p>
      {data.map((users, index) => {
        // `data`は、各ページのAPIレスポンスの配列です。
        return users.map(user => <div key={user.id}>{user.name}</div>)
      })}
      <button onClick={() => setSize(size + 1)}>Load More</button>
    </div>
  )
}
```

`getKey()` は、`useSWR()` との大きな違いです。現在のページのインデックスと前のページデータを受け取ります。そのため、インデックスベースとカーソルベースの両方のページネーション API を適切にサポートできます。

上記のソースコードが受け取る値(fetcher が返す値)は、以下のようになっている事に注意してください。

```json5:公式サイトより引用
GET '/users?page=0&limit=10'
[
  { name: 'Alice', ... },
  { name: 'Bob', ... },
  { name: 'Cathy', ... },
  ...
]
```

そのため `useSWRInfinite()` の `data` の結果は、以下のようになっています。

```ts:公式サイトより引用
// `data` will look like this
[
  [
    { name: 'Alice', ... },
    { name: 'Bob', ... },
    { name: 'Cathy', ... },
    ...
  ],
  [
    { name: 'John', ... },
    { name: 'Paul', ... },
    { name: 'George', ... },
    ...
  ],
  ...
]
```

これでページネーション機能の紹介は終わりです。シンプル過ぎて、特に解説するところもなかったですね 😅

# 自動再検証( Auto Revalidation )

自動再検証( Auto Revalidation )について解説したいと思います。

SWR と言う名前は、[stale-while-revalidate](https://tools.ietf.org/html/rfc5861) の頭文字から来ていると言いましたが、この [stale-while-revalidate](https://tools.ietf.org/html/rfc5861) にはキャッシュを更新する必要があるのかを**検証する**プロセスがあります。勿論、SWR も [stale-while-revalidate](https://tools.ietf.org/html/rfc5861) を踏まえた設計になっているので、検証をするのですが、SWR では自動で検証するようになっており、その検証タイミングも様々なモノがあります。

この節では、どのタイミングで自動再検証が行われているのかを見て行きたいと思います。

## フォーカス時に再検証する( Revalidate on Focus )

ページがフォーカスした時、タブを切り替えた時、または`focusThrottleInterval`オプションで指定した期間内( ポーリングする時間 )に、SWR は自動的にデータを再検証します。

これにより、画面の内容を常に最新の状態に保つことができるので、サービスのユーザー体験を高めることができます。シナリオとしては、スリープ状態になった PC が復帰した時などに役立つと思います。

具体的な挙動については以下の公式サイトに動画がありますので、是非見て頂けると動作内容を把握できると思います。

https://swr.vercel.app/docs/revalidation#revalidate-on-focus

:::message
この機能はデフォルトでは**有効**になっており、 `revaildateOnFocus` オプションで無効にすることができます。
:::

:::message
`focusThrottleInterval`オプションで、検証する時間を変更することができます。
デフォルトは`5000ms`に設定されています。
:::

## 秒間隔で再検証する( Revalidate on Interval )

画面上のデータを時間と共に更新したい時、SWR の `refreshInterval` オプションを設定する事で、それが可能になります。

```ts:refreshIntervalを設定
// 1秒ごとに再検証する
useSWR('/api/todos', fetcher, { refreshInterval: 1000 })
```

これも具体的な処理内容は、以下の公式サイトの動画見て頂けると分かると思います。

https://swr.vercel.app/docs/revalidation#revalidate-on-interval

注意として、この時に場合によっては通信処理が多く発生する可能性があります！そのため、サーバーへの負担が大きくなることが予想されます。なので、そこまでリアルタイム性を求めていないのであれば、この機能は無効にしてもいいかもしれません。

:::message
この機能はデフォルトでは**無効**となっています。
この機能が有効でも、デフォルトでは Web ページが画面に表示されていない時、またはネットワーク接続がない時は再検証されません。再検証するには、`refreshWhenHidden` または `refreshWhenOffline` オプションを有効にする必要があります。
:::

## 再接続時に再検証する( Revalidate on Reconnect )

ユーザーがオンラインに復帰した時に再検証することも可能です。

このシナリオは、ユーザーの PC がロックを解除した時に頻繁に発生しますが、正直これが本当に必要なモノなのかはサービスによると思いますが、多くのサービスでは必要ないかもしれません。

`revalidateOnReconnect` オプションで設定することができます。

```ts:revalidateOnReconnectを設定
useSWR("/api/user", fetcher, { revalidateOnReconnect: true })
```

※ この機能は `navigator.onLine` によってオンラインに復帰したかどうかを判断しています。

:::message
この機能はデフォルトで**有効**になっています。
:::

:::message
現在(2021/06 時点)、[Chrome のバグ](https://bugs.chromium.org/p/chromium/issues/detail?id=678075)があり `navigator.onLine` を使ってブラウザが「オンライン」か「オフライン」かを検出することは信頼できません。
回避策として、SWR では最初のロード時には常にオンラインであると想定し、「オンライン」または「オフライン」イベント時にステータスを変更します。
参照: [SWR のソースコードより引用](https://github.com/vercel/swr/blob/e3d42488d7457019f5f9ca3b6e03357c8dd33130/src/utils/web-preset.ts#L4-L10)
:::

## マウント時に再検証する( Revalidate on Mount )

`useSWR()` を実行しているコンポーネントがマウントした時にも、再検証することが可能です。

`revalidateOnMount` オプションを設定する事ができます。

```ts
useSWR("/api/user", fetecher, { revalidateOnMount: true });
```

:::message
デフォルトでは `fallbackData` オプションが設定されていない場合、**マウント時に再検証が行われます。**
:::

## 自動再検証を行いたくない場合

プロジェクトによっては、上記すべての再検証を行いたくない場合があります。
そのような場合には、 `useSWRImmutable()` を使うことで、すべての自動再検証を無効にできます 👇

```ts:自動再検証の無効化
import useSWRImmutable from 'swr/immutable'

useSWRImmutable(key, fetcher, options)

// useSWRImmutable()は、以下と同等です。
useSWR(key, fetcher, {
  revalidateIfStale: false,
  revalidateOnFocus: false,
  revalidateOnReconnect: false
})
```

基本的には、`useSWR()`と同じオプションや挙動をするので、`useSWR()`の[糖衣構文](https://ja.wikipedia.org/wiki/糖衣構文)として使えます。

# プリフェッチ(Prefetching)

上記で解説した、[Mutation](#mutation) を使う事でデータを予めプリフェッチすることができます。

```ts:公式サイトから引用
import { mutate } from 'swr'

function prefetch () {
  mutate('/api/data', fetch('/api/data').then(res => res.json()))
  // the second parameter is a Promise
  // SWR will use the result when it resolves
}
```

しかし、公式サイトによると一番オススメなのは Top-Level でやる事らしいので、こちらが使えるなら、Top-Level の方を使った方が良いと思います。以下は[公式サイト](https://swr.vercel.app/docs/prefetching)からの引用＆要約です。

> SWR のデータをプリフェッチする方法はたくさんあります。トップレベルのリクエストに rel="preload"を指定する方法は、強くお勧めします。
>
> ```html
> <link rel="preload" href="/api/data" as="fetch" crossorigin="anonymous" />
> ```
>
> HTML の\<head\>内に配置するだけです。簡単、高速、ネイティブです。
>
> JavaScript がダウンロードを開始する前であっても、HTML がロードされるときにデータをプリフェッチします。 同じ URL を使用するすべての着信フェッチ要求は、結果を再利用します（もちろん、SWR を含む）。

## 初期値を設定する

SSR や SSG を使用している時などでは、`useSWR()`でリクエストする前に既にデータを取得していることがあります。そのような場合には、`fallbackData`オプションを使うことで、あらかじめ初期値を設定することができます。

```ts:初期値を設定する
// "hoge"を`'/api/data'`の初期値として設定する
useSWR('/api/data', fetcher, { fallbackData: "hoge" })
```

また、`<SWRConfig />`の`fallback`オプションを使うことで、より広範囲かつ柔軟に初期値を設定できます 👇

```tsx:公式ブログより引用
<SWRConfig value={{
  fallback: {
    '/api/user': { name: 'Bob', ...  },
    '/api/items': {...},
    ...
  }
}}>
  <App/>
</SWRConfig>
```

# Dependency Collection について( 重要 )

Dependency Collection について解説したいと思いますが、**これは結構重要です。**
なので、SWR を使う人は是非とも知っておきたい知識となっていますので、気合入れて読んでください 🥋
まず、[公式サイト](https://swr.vercel.app/advanced/performance#dependency-collection)の内容を引用＆要約を以下に記述します。

> `useSWR` が返す３つの**ステートフルな値**: `data`, `error` ,`isValidating`は、 それぞれが独立して更新することができます。例えば、完全なデータフェッチライフサイクル内でこれらの値を出力すると、次のようになります:
>
> ```ts:data,error,isValidatingを取得してconsole.logで表示
> function App () {
>   const { data, error, isValidating } = useSWR('/api', fetcher)
>   console.log(data, error, isValidating)
>   return null
> }
> ```
>
> 最悪の場合（最初の要求が失敗し、次に再試行が成功した）、4 行のログが表示されます。
>
> ```ts:console.logの結果
> // console.log(data, error, isValidating)
> undefined undefined true  // => フェッチの開始
> undefined Error false     // => フェッチの完了、エラーを取得
> undefined Error true      // => 再試行の開始
> Data undefined false      // => 再試行の完了、データを取得
> ```
>
> この状態変化は理にかなっています。しかし、それはまた、コンポーネントが 4 回レンダリングされることを意味しています。
>
> コンポーネントを変更して `data` だけを使用する場合：
>
> ```ts:dataだけ取得するように修正(fetcherの挙動は変化してない)
> function App () {
>   const { data } = useSWR('/api', fetcher)
>   console.log(data)
>   return null
> }
> ```
>
> 魔法が起こります — 今回は**二つの再レンダリング**しかありません：
>
> ```ts:console.logの結果
> // console.log(data)
> undefined // => 再利用 / 初期レンダリング
> Data      // => 再試行の完了、データを取得
> ```
>
> まったく同じプロセスが内部で発生し、最初のリクエストからエラーが発生し、再試行からデータを取得しました。ただし、**SWR はコンポーネントによって使用されている状態のみを更新します。** 今回の例の場合は、`data` のみ。

上記の内容では、`useSWR()` から**受け取る値を変更するだけで、コンポーネントの描画更新が変化しています。**
なぜこのような事が起こるのかと言うと、SWR 側が値を取得しているかを検知して、最適な描画更新をしているためです。具体的には以下のソースコードです。

※ [公式リポジトリのソースコード](https://github.com/vercel/swr/blob/e3d42488d7457019f5f9ca3b6e03357c8dd33130/src/use-swr.ts#L475-L489)より引用。

```ts:swrのソースコードから抜粋
return {
  mutate: boundMutate,
  get data() {
    stateDependencies.data = true
    return data
  },
  get error() {
    stateDependencies.error = true
    return error
  },
  get isValidating() {
    stateDependencies.isValidating = true
    return isValidating
  }
} as SWRResponse<Data, Error>
```

上記のソースコードは、[getter](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Functions/get) を利用して、値が使われているかを検知して、それぞれを **ステートフルな値** として扱っています。

仕組みは意外と簡単なのですが、初見時にちょっと驚いたのは内緒です 🤫

この挙動を知らずに、使ってもないのに `error` や `isValidating` を取得してると、**無駄な描画更新が発生してしまうので注意しましょう！**

# Suspence Mode

:::message alert
Suspence Mode は現在(2021/01 時点)、React の**実験的な**機能です。これらの API は、React の一部になる前に、警告なしに大幅に変更される可能性があります。
詳しくは[こちら](https://ja.reactjs.org/docs/concurrent-mode-suspense.html)を参考にして下さい。
:::

:::message
ReactSuspense は SSR モードではまだサポートされていないことに注意してください。
:::
`suspence` オプションを有効にすることで Suspence Mode を有効にすることができます。

```ts:公式サイトより引用したものを少し改変
import { Suspense } from 'react'
import useSWR from 'swr'

function Profile () {
  const { data } = useSWR('/api/user', fetcher, { suspense: true }) // suspenceを有効に設定！
  return <div>hello, {data.name}</div>
}

function App () {
  return (
    <Suspense fallback={<div>loading...</div>}>
      <Profile/>
    </Suspense>
  )
}
```

:::message
`suspense` オプションはライフサイクルで変更できないことに注意してください。
:::

Suspence Mode では、`useSWR()` が返す `data` は、常に通信結果になります(`undefined`は含まれない)。正し、エラーが発生した場合は、[Error Boundary](https://ja.reactjs.org/docs/error-boundaries.html) を使用してエラーをキャッチする必要があります。

```tsx:公式サイトより引用
<ErrorBoundary fallback={<h2>Could not fetch posts.</h2>}>
  <Suspense fallback={<h1>Loading posts...</h1>}>
    <Profile />
  </Suspense>
</ErrorBoundary>
```

## Note:条件付きフェッチを使用する場合

通常、`suspence` オプションを有効にすると、レンダリング時に `data` が準備されている事が保証されています。

しかし、条件付きフェッチ又は依存フェッチと一緒に使用すると、要求が**一時停止された**場合に `data` が `undefined` になります。

```ts:公式サイトより引用
function Profile () {
  const { data } = useSWR(isReady ? '/api/user' : null, fetcher, { suspense: true })
  // `data` will be `undefined` if `isReady` is false
  // ...
}
```

この制限に関する技術的な詳細が知りたい方は、こちらの[Discussion](https://github.com/vercel/swr/pull/357#issuecomment-627089889)を参照してください。

# Next.js との連携( SSG/SSR 対応 )

Next.js では、SSG と SSR という強力な機能を使うことができます。SWR はそれらの機能と**一緒に使うことができます。**

具体的には以下のようにします 👇

```ts:公式サイトより引用したものを少し改変
export async function getStaticProps () {
  // `getStaticProps` はサーバー側で実行されます
  const article = await getArticleFromAPI()
  return {
    props: {
      fallback: {
        '/api/article': article // '/api/article' の初期値を返す
      }
    }
  }
}

export default function Page({ fallback }) {
  // getStaticProps()で受け取った初期値を反映する
  return (
    <SWRConfig value={{ fallback }}>
      <Article />
    </SWRConfig>
  )
}

function Article() {
  // `data` の中には <SWRConfig /> で設定された初期値が渡される
  const { data } = useSWR('/api/article', fetcher)
  return <h1>{data.title}</h1>
}
```

これにより、初期レンダリングを早めつつ、それ以降の動作も SWR によって、動的に素早く動作させることができます。

また、個別に初期値を設定したい場合は、`fallbackData`を使うと便利です 👇

```ts:fallbackDataを使う例
useSWR('/api/article', fetcher, { fallbackData: articleValue })
```

# Middleware について

:::message
Middleware を多用すると処理が複雑化してバグを発生させやすいため、使用を控えることをオススメします。
:::

SWR では、デフォルトの挙動を拡張できる Middleware を設定できます。
具体的には以下のような関数です 👇

```ts:公式サイトより引用
function myMiddleware (useSWRNext) {
  return (key, fetcher, config) => {
    // フックが実行される前...

    // 次のミドルウェア、またはこれが最後のミドルウェアの場合は `useSWR` を処理します。
    const swr = useSWRNext(key, fetcher, config)

    // フックが実行された後...
    return swr
  }
}
```

上記の関数を `use` オプションに渡すことで、`useSWR()` の挙動を拡張できます 👇

```tsx:公式サイトから引用したモノを少し改変
<SWRConfig value={{ use: [myMiddleware] }}>
  {/* ... */}
</SWRConfig>

// または...

useSWR(key, fetcher, { use: [myMiddleware] })
```

注意点として、 Middleware は実行順に継承されていくため、以下の二つの処理は同じになります 👇

```tsx:公式サイトより引用
function Bar () {
  useSWR(key, fetcher, { use: [c] })
  // ...
}

function Foo() {
  return (
    <SWRConfig value={{ use: [a] }}>
      <SWRConfig value={{ use: [b] }}>
        <Bar/>
      </SWRConfig>
    </SWRConfig>
  )
}
```

```ts:公式サイトより引用
useSWR(key, fetcher, { use: [a, b, c] })
```

## 実行順について

Middleware の理解を深めるために、実際に Middleware の実行順を確認していきましょう。
まずは、以下のような処理を書いたとします 👇

```ts:公式サイトより引用
useSWR(key, fetcher, { use: [a, b, c] })
```

この時の実行順は、以下のようになります 👇

```ts:公式サイトより引用
enter a
  enter b
    enter c
      useSWR()
    exit  c
  exit  b
exit  a
```

上記から分かるように、`a → b → c → useSWR()` の順で実行されています。
これはつまり、もし `a` の部分にバグがあり処理が中断した場合は、それ以降の `b → c → useSWR()` は実行されないことを意味します。そのため、**Middleware の順番は重要**であり、**Middleware のバグは大きなバグに繋がりやすい**ことを理解しておく必要があります。

## 具体例について

一番身近な具体例として、上記で解説した `useSWRInfinite()` が挙げられます 👇

https://github.com/vercel/swr/blob/ec778e70c8e17c239f2837c14dea99ce9a429fb0/infinite/index.ts

また、公式ドキュメントには他の Middleware の実装例も紹介されていますので、Middleware を自作する際の参考にするといいと思います 👇

https://swr.vercel.app/ja/docs/middleware#%E4%BE%8B

# キャッシュについて

:::message
**キャッシュを扱う際には注意して扱ってください！**
基本的にキャッシュを扱うような処理は、複雑なバグを引き起こす原因になります。
:::

SWR では、Cache Provider を使用してキャッシュの処理をカスタマイズすることができますが、Cache Provider として使用するには、以下の型を満たすようなオブジェクトである必要があります 👇

```ts:公式サイトより引用
interface Cache<Data> {
  get(key: string): Data | undefined
  set(key: string, value: Data): void
  delete(key: string): void
}
```

※ 上記の型を満たすものとして [JavaScript の Map](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Map) などがあります。

## Cache Provider の設定

Cache Provider を設定するには、`<SWRConfig />` を使用します。

```tsx:公式サイトより引用
import useSWR, { SWRConfig } from 'swr'

function App() {
  return (
    <SWRConfig value={{ provider: () => new Map() }}>
      <Page/>
    </SWRConfig>
  )
}
```

またこの時、 `provider` に渡す関数の第一引数には、親のコンポーネントの Cache Provider ( 無い場合はデフォルトのキャッシュ )を受け取る事ができるため、それを使って拡張する事もできます 👇

```tsx:公式サイトより引用
<SWRConfig value={{ provider: (cache) => newCache }}>
  ...
</SWRConfig>
```

:::message
この機能は実験的なモノであり、将来のアップグレードで動作が変更される可能性がある事に注意してください！
:::

## Cache Provider の取得

カスタマイズした Cache Provider は、`useSWRConfig()` で取得することができます 👇

```tsx:カスタマイズしたcacheを取得する
import { useSWRConfig } from 'swr'

const Component = () => {
  const { cache } = useSWRConfig()

  const value = cache.get("/api/users") // "/api/users"のキャッシュを取得する
  const onClear = () => cache.clear()   // すべてのキャッシュを削除

  // ...
}
```

## 具体的な処理について

公式サイトにいくつかの具体例がありますので、そちらを参考にして頂けたらと思います 👇

https://swr.vercel.app/ja/docs/advanced/cache#例

# React Native への対応

SWR は、[React Native](https://reactnative.dev/) でも使用することができます。
基本的な使い方は同じですが、`focus` や `online` といった**ブラウザ特有のイベントを React Native 用に設定しなおす必要があります**👇

```tsx:公式サイトより引用
<SWRConfig
  value={{
    provider: () => new Map(),
    isOnline() {
      /* ネットワーク状態の検出器をカスタマイズ */
      return true
    },
    isVisible() {
      /* 視認状態の検出器をカスタマイズ */
      return true
    },
    initFocus(callback) {
      /* リスナーをステートプロバイダーに登録 */
    },
    initReconnect(callback) {
      /* リスナーをステートプロバイダーに登録 */
    }
  }}
>
  <App />
</SWRConfig>
```

`isOnline` と `isVisible` は、アプリが "アクティブ" かどうかを判断するための関数です。デフォルトでは、上記二つが "アクティブ" でない場合、 SWR は再検証を中止します。

また、 `initFocus` と `initReconnect` を使用するには、前述した Cache Provider も設定する必要があります。

上記の設定について、`initFocus` の実装については、公式ドキュメントに具体例がありましたので、以下にソースコードを引用します 👇

```tsx:公式サイトより引用
<SWRConfig
  value={{
    provider: () => new Map(),
    isVisible: () => { return true },
    initFocus(callback) {
      let appState = AppState.currentState

      const onAppStateChange = (nextAppState) => {
        /* バックグラウンドモードまたは非アクティブモードからアクティブモードに再開する場合 */
        if (appState.match(/inactive|background/) && nextAppState === 'active') {
          callback()
        }
        appState = nextAppState
      }

      // アプリの状態変更を監視する
      const subscription = AppState.addEventListener('change', onAppStateChange)

      return () => {
        subscription.remove()
      }
    }
  }}
>
  <App>
</SWRConfig>
```

上記のソースコードにより、React Native でもアクティブモードへ移行した時に、フォーカス時の再検証( Revalidate on Focus )が実行されるようになります。

# オプションについて

`useSWR`にはオプションを渡すことができます。

```ts:useSWRにオプションを渡す例
const { data } = useSWR(key, fetcher, optoins);
```

この`options`の内容をご紹介したいと思いますが、結構量があるため、簡単な解説と型情報をのみを記述したいと思います。

:::details オプション一覧

## suspense

| 型      | デフォルト値 | 効果                                                                                               |
| ------- | ------------ | -------------------------------------------------------------------------------------------------- |
| boolean | false        | [React Suspence モード](https://ja.reactjs.org/docs/concurrent-mode-suspense.html)を有効にします。 |

## fetcher

| 型        | デフォルト値 | 効果                                  |
| --------- | ------------ | ------------------------------------- |
| ※以下参照 | undefined    | デフォルトの`fetcher`関数を設定します |

```ts:fetcherの型
type Fetcher<Data> = (...args: any) => Data | Promise<Data>
```

## fallback

| 型  | デフォルト値 | 効果                                |
| --- | ------------ | ----------------------------------- |
| any | undefined    | 初期データの key-value オブジェクト |

## fallbackData

| 型  | デフォルト値 | 効果               |
| --- | ------------ | ------------------ |
| any | undefined    | 初期値を設定します |

## revalidateIfStale

| 型      | デフォルト値 | 効果                                                   |
| ------- | ------------ | ------------------------------------------------------ |
| boolean | true         | 古いデータがある場合でも、マウント時に自動再検証をする |

## revalidateOnMount

| 型      | デフォルト値 | 効果                                               |
| ------- | ------------ | -------------------------------------------------- |
| boolean | ※以下参照    | コンポーネントがマウントした時に自動的に再検証する |

※ デフォルトでは `fallbackData` が**設定されてない場合**、マウント時に再検証されます。`false` だと `fallbackData` を設定していても、再検証されません。

## revalidateOnFocus

| 型      | デフォルト値 | 効果                                                 |
| ------- | ------------ | ---------------------------------------------------- |
| boolean | true         | ウィンドウがフォーカスされたときに自動的に再検証する |

※ `focusThrottleInterval` オプションで検証する期間を変更することができます。

## revalidateOnReconnect

| 型      | デフォルト値 | 効果                                                       |
| ------- | ------------ | ---------------------------------------------------------- |
| boolean | true         | ブラウザがネットワーク接続を回復した時に自動的に再検証する |

## refreshInterval

| 型     | デフォルト値 | 効果                                         |
| ------ | ------------ | -------------------------------------------- |
| number | 0            | ポーリング間隔のミリ秒（デフォルトでは無効） |

## refreshWhenHidden

| 型      | デフォルト値 | 効果                                                              |
| ------- | ------------ | ----------------------------------------------------------------- |
| boolean | false        | ウィンドウが非表示の時にポーリングする (`navigator.onLine`で判断) |

※ `refreshInterval` が有効になっている場合にのみ動作します。`true` に設定する時は、**必ず** `refreshInterval` を設定してください。

## refreshWhenOffline

| 型      | デフォルト値 | 効果                                                                |
| ------- | ------------ | ------------------------------------------------------------------- |
| boolean | false        | ブラウザがオフラインの時にポーリングする (`navigator.onLine`で判断) |

## refreshWhenOffline

| 型      | デフォルト値 | 効果                                                                |
| ------- | ------------ | ------------------------------------------------------------------- |
| boolean | false        | ブラウザがオフラインの時にポーリングする (`navigator.onLine`で判断) |

## shouldRetryOnError

| 型      | デフォルト値 | 効果                                       |
| ------- | ------------ | ------------------------------------------ |
| boolean | true         | fetcher にエラーが発生したときに再試行する |

## dedupingInterval

| 型     | デフォルト値 | 効果                                                 |
| ------ | ------------ | ---------------------------------------------------- |
| number | 2000         | この期間内に同じキーでのリクエストの重複を排除します |

## focusThrottleInterval

| 型     | デフォルト値 | 効果                     |
| ------ | ------------ | ------------------------ |
| number | 5000         | 再検証する期間を指定する |

※ ここでの再検証とは、`revalidateOnFocus`の事を指します。`revalidateOnFocus`オプションが false の場合は、このオプションは機能しません。

## loadingTimeout

| 型     | デフォルト値 | 効果                                                    |
| ------ | ------------ | ------------------------------------------------------- |
| number | 3000         | `onLoadingSlow`イベントをトリガーするためのタイムアウト |

※ 低速ネットワーク（2G、<= 70Kbps）の場合、`loadingTimeout`は 5 秒になります。

## errorRetryInterval

| 型     | デフォルト値 | 効果                             |
| ------ | ------------ | -------------------------------- |
| number | 5000         | エラーが発生した時の再試行の間隔 |

※ 低速ネットワーク（2G、<= 70Kbps）の場合、`errorRetryInterval`は 10 秒になります。

## errorRetryCount

| 型     | デフォルト値 | 効果                 |
| ------ | ------------ | -------------------- |
| number | ※以下参照    | 最大エラー再試行回数 |

※ デフォルトでは、[exponential backoff アルゴリズム](https://en.wikipedia.org/wiki/Exponential_backoff)を使用してエラーの再試行を処理します

## onLoadingSlow

| 型        | デフォルト値 | 効果                                                           |
| --------- | ------------ | -------------------------------------------------------------- |
| ※以下参照 | undefined    | リクエストの読み込みに時間がかかりすぎる場合のコールバック関数 |

```ts:onLoadingSlowの型
// SWROptions は、useSWRの第三引数に渡したオプションのオブジェクトです。
type onLoadingSlow = (key: string, config: SWROptions) => void
```

## onSuccess

| 型        | デフォルト値 | 効果                                             |
| --------- | ------------ | ------------------------------------------------ |
| ※以下参照 | undefined    | リクエストが正常に終了したときのコールバック関数 |

```ts:onSuccessの型
// SWROptions は、useSWRの第三引数に渡したオプションのオブジェクトです。
type onSuccess = (data: any, key: string, config: SWROptions) => void
```

## onError

| 型        | デフォルト値 | 効果                                             |
| --------- | ------------ | ------------------------------------------------ |
| ※以下参照 | undefined    | リクエストがエラーを返したときのコールバック関数 |

```ts:onErrorの型
// SWROptions は、useSWRの第三引数に渡したオプションのオブジェクトです。
// ※ error は fetcher が reject した値です
type onError = (error: any, key: string, config: SWROptions) => void
```

## onErrorRetry

| 型        | デフォルト値 | 効果                               |
| --------- | ------------ | ---------------------------------- |
| ※以下参照 | undefined    | エラー時の再試行をするコールバック |

```ts:onErrorRetryの型
// SWROptions は、useSWRの第三引数に渡したオプションのオブジェクトです。
// ※ error は fetcher が reject した値です
type onErrorRetry = (
  error : any,
  key: string,
  config: SWROptions,
  revalidate: (options: RevalidateOptions) => Promise<boolean>,
  revalidateOptions: RevalidateOptions
) => void

// 再検証のオプション型
type RevalidateOptions = {
  dedupe: boolean;
  retryCount: number;
}
```

## compare

| 型        | デフォルト値 | 効果                                                                                                     |
| --------- | ------------ | -------------------------------------------------------------------------------------------------------- |
| ※以下参照 | ※以下参照    | 誤った再レンダリングを回避するために、返されたデータがいつ変更されたかを検出するために使用される比較関数 |

```ts:compareの型
type compare = (a: any, b: any) => boolean
```

※ デフォルト値では、[dequal](https://github.com/lukeed/dequal)が使われています

## isPaused

| 型        | デフォルト値 | 効果                                       |
| --------- | ------------ | ------------------------------------------ |
| ※以下参照 | () => false  | 再検証を一時停止するかどうかを検出する関数 |

```ts:isPausedの型
type isPaused = () => boolean
```

※ `isPaused` が `true` を返す時、再検証を停止し、フェッチされたデータとエラーを無視します。デフォルトでは、`false` を返します。

詳細は以下のプルリクを参照してください 🏳‍🌈

https://github.com/vercel/swr/pull/845

## use

| 型        | デフォルト値 | 効果                   |
| --------- | ------------ | ---------------------- |
| ※以下参照 | undefined    | ミドルウェア関数の配列 |

```ts:useの型
type Middleware = (useSWRNext: SWRHook) => SWRHookWithMiddleware

type use = Middleware[]
```

:::

# あとがき

ここまで[SWR](https://swr.vercel.app/)の基本的な使い方から、詳細な挙動について解説してきました。

[SWR](https://swr.vercel.app/)はとてもシンプルなうえに、複雑な非同期処理を肩代わりしてくれるので、使っていてとても楽しいですね。これからもどんどん使って行きたいと思いました 💪

もし記事の内容に何か不備等がございましたら、コメントなどで教えて頂けると幸いです 🙏

あっ、あと言い忘れていましたが、記事のタイトルは[SWR](https://swr.vercel.app/)をリスペクトして名付けました。どうでもいいですね 🙄

ここまで読んでくれてありがとうございます。
これが誰かの参考になれば幸いです。
それではまた 👋
