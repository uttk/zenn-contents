---
title: "もしも React Hooks がコンポーネントの外に羽ばたいたら"
emoji: "🛸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, typescript]
published: true
---


## この記事について

開発していて、「 この API はこうだったらいいな～ 」って思うことはありますか？
私はよく思っちゃうほうで、最近の開発でも使っている API について考えてしまって、夜も寝れなくなっちゃうことがあります（嘘）。

普段ならそんなくだらないことは自分の頭の中だけで食い散らかすだけなんですが、たまには私の頭の中のアイディアを文章にして、少しでも文章力を上げようかな～と思い、

「 **もしも React Hooks がコンポーネントの外に書けるようになったら？** 」

というテーマで今回記事を書いていこうかなと思います🪀 ~~別に記事のネタが思い浮かばなかったってわけじゃないよ~~ 


## React Hooks について思うこと

さて、みなさんは React Hooks は好きですか？私は好きです。
今までクラスで書いていた複雑な状態管理が、 React Hooks の登場によって実装しやすくなった印象があります。

しかし、React Hooks について一つだけ思うところがあります。
それは**コンポーネントの中に書かなければならない**ということです👇

```tsx:なぜコンポーネントの中に書かなければならないのだ！
const Component = () => {
  const [state, setState] = useState(0)

  return /*...*/
}
```

コンポーネントの中にしか書けないので、描画部分とロジックが分離しにくかったり、テストがやりにくいなどの問題があります。

実際、みなさんは以下のように肥大化してしまったコンポーネントを見たことはありませんか？私はよくあります。(なお、書いているのは過去の自分😇)

```tsx:肥大化してしまったコンポーネント例
const Component = () => {
  // 👇 増えてしまったuseState
  const [state, setState] = useState(/*...*/)
  const [state2, setState2] = useState(/*...*/)
  const [state3, setState3] = useState(/*...*/)

  // 👇 増えてしまったカスタムHooks
  const anyData = useCustomHooks(/*...*/)
  const anyData2 = useCustomHooks2(/*...*/)
  const anyData3 = useCustomHooks3(/*...*/)

  // 👇 増えてしまったメモ化
  const memoData = useMemo(/*...*/, [])
  const memoData2 = useMemo(/*...*/,[/*...*/])
  const memoData3 = useMemo(/*...*/,[/*...*/])

  // 👇 増えてしまったイベントハンドラー
  const handleEvent = useCallback(() => {
    /* めっちゃ長いコード */
  }, [/*...*/])
  const handleEvent2 = useCallback(() => {
    /* めっちゃ長いコード */
  }, [/*...*/])
  const handleEvent3 = useCallback(() => {
    /* めっちゃ長いコード */
  }, [/*...*/])

  // 👇 増えてしまったuseEffect
  useEffect(() => {/*...*/}, [])
  useEffect(() => {/*...*/}, [/*...*/])
  useEffect(() => {/*...*/}, [/*...*/])

  return (
    /* とどめにめちゃくちゃ長いJSX🥺 */
  )
}
```

上の例はちょっと大げさですが、似たようなコンポーネントはどのプロジェクトでも発生していると思います。

「いやいや、コンポーネント設計が悪いだけじゃないの？」って言われたらそれはそうなんですが、フロントエンドだと JSX の部分が長くなってしまうことがあるので、やっぱり描画(JSX)部分とロジック(Hooks)部分を完全に分けて実装したいっていうのがあります。[^1]

[^1]: あと、無闇にコンポーネントを量産したくない

なので、理想としては Hooks はコンポーネントに書かずに何とかして外に書いて、引数から State などを受け取るようにしたいわけです👇

```tsx:コンポーネントはPropsを受け取って表示するだけが理想
interface PropsType {
  // Props から State とかを受け取りたい
  state: [any, SetState];
  handleEvent: (event: any) => void;
}

const Component = (props: PropsType) => {
  return (
    /* めちゃくちゃ長いJSX */
  )
}
```


## 新しい API を設計してみる

はい、そんなこんなで新しい API を設計したいと思います。
まずは要件を整理します。ざっと洗い出すと以下のような感じになりました👇

- [フックのルール](https://ja.legacy.reactjs.org/docs/hooks-rules.html)を必要としない
- 関数型っぽくかける(雰囲気だけ)
- React Hooks できることは全部できるようにする
- React Hooks と併用できるようにする
- せっかくだから関数コンポーネントでもクラスコンポーネントでも使用できるようにする

特に変な要件はないと信じたいですが、[フックのルール](https://ja.legacy.reactjs.org/docs/hooks-rules.html) については主に [フックを呼び出すのはトップレベルのみ](https://ja.legacy.reactjs.org/docs/hooks-rules.html#only-call-hooks-at-the-top-level) のことを言っています。このルールは実装上のメリットもありますが、実装方法を強制してしまうのは良くないかなと個人的に思っているので、必要としないようにしています。


さて、これから上記の要件を満たすように API を設計していきますが、これから設計する API に名前がないと不便なので、**この記事ではとりあえず `fly` という名前を付けることとします。**

ちなみに、名前は適当です🧃

### 基本的な使い方を考えてみる

まずはローカルステートの実装を考えてみます。
以下のソースコードは fly を使ってローカルステートを実装してみた例です 👇

```tsx:基本的な使い方(ローカルステート)を実装してみる
import { createRoot } from 'react-dom/client';
import { fly, applyFly, type GetFlyType } from "xxx"

const flyCounter = fly(
  // この関数の返り値が第二引数で指定したコンポーネントにPropsとして渡されます
  ({ createState }) => {
    const [counter, setCounter] = createState(0);
    return { counter, setCounter }
  },  
  // fly を props で受け取りたいコンポーネントはここの配列に定義します
  [ AppComponent ] 
)

interface AppProps {
  // ここの Props 名は`root.render()`に渡した変数名になります
  flyCounter: GetFlyType<typeof flyCounter>
}

function AppComponent({ flyCounter }: AppProps) {
  const { counter, setCounter } = flyCounter
  return (
    <>
      <p>click now: {counter}</p>
      <button onClick={() => setCounter(counter + 1)}>increment</button>
    </>
  )
}

/* ------- 以下の処理は仮実装です！もっといい方法を考え中！ ------- */

const App = applyFly(
  AppComponent, 
  [flyCounter] // 使用する fly をここに指定します
)

const root = createRoot(document.getElementById('app'));
root.render(<App />);
```

基本的な実装は `fly()` によってコンポーネントに渡す Props を定義しています。もちろんコンポーネントの外なので、[フックのルール](https://ja.legacy.reactjs.org/docs/hooks-rules.html)は適用されないため、if 文なども State 宣言時に使用することができます👇

```tsx:flyではState宣言時にif文なども使える
const flyCounter = fly(
  ({ createState }) => {
    if(IS_DEV === true) { // if 文が使える！ 
      const [counter, setCounter] = createState(0);
      return { counter, setCounter }
    }
    return {}
  },
  [ AppComponent ]
);
```

また、`fly()` は定義時に使用するコンポーネントを指定する必要があります👇

```ts:flyは使用するコンポーネントを定義時に指定する必要がある
const flyAny = fly(
  () => {/*...*/},
  [ AppComponent ] // コンポーネントを指定する必要がある！
);
```

これには二つの理由があります。

一つ目は、**自動的に Props に値を渡すため** です。
fly で定義した値を受け取るコンポーネントは必然的に Props から受け取るため、もし明示的に渡すようにするとコンポーネントの利便性が落ちてしまいます👇

```tsx:flyに依存するコンポーネントを使うたびにPropsを指定する必要があり面倒
function AppComponent() {
  return (
    <AnyComponent 
      flyAny={flyAny}  
      flyAny2={flyAny2}  
      flyAny3={flyAny3}  
      ...
      onSubmit={() => {/*...*/}}
    />
  )
}
```

そのため fly では、明示的に渡さなくても済むように Props を省略できるようにしています👇

```tsx:flyによって自動的にStateが渡されるので指定する必要が無い
function AppComponent() {
  return (
    <AnyComponent onSubmit={() => {/*...*/}} />
  )
}
```

ただし、Props に明示的に `fly()` の返り値を渡しても問題ありません。あくまで Props がオプショナルになるだけですので、省略するかどうかは開発者が判断する感じになると思います。


そして二つ目は、**影響範囲を把握するため** です。
ある State 依存するコンポーネントが多くなりすぎて迂闊に変更できない、または変更するのに多くの工数が必要になってしまう、というのはフロントエンドエンジニアの方なら誰しもが経験すると思います。

fly ではそのようなことが無いように、宣言時にコンポーネントを指定させることで、影響範囲を可視化し管理しやすくすると共に、fly が更新処理を行う時の最適化処理にも使用できるようにするという一石二鳥(？)の設計になっています。

また、Dev Tools やビルド時の最適化にも使用できると思うので、影響範囲を明記してもらった方が色々とメリットがあると思ってこの設計にしました。

### 別の fly の値を使う

次は fly 同士の連携について考えます。
React Custom Hooks では、Custom Hooks の中で Custom Hooks を使うことができますが、fly でも同じことをできるようにします👇

```tsx
import { fly } from "xxx"

export const flyAny = fly(/*...*/, [ AppComponent ])

export const flyCounter = fly(
  ({ getFly }) => {
    const {/*...*/} = getFly(flyAny); // 別の fly の値を取得できる

    /* flyAny の値を使った処理 */
  },  
  [ AppComponent ] 
)
```

引数から渡される `getFly()` を使うことで、fly の中で別の fly を使うことができます。 

注意点としては、`getFly()` で取得する fly でも使用するコンポーネントを指定する必要がありますので、もし指定して無い場合はエラーになります👇

```tsx:getFly()がエラーになる例
export const flyAny = fly(/*...*/, [])

export const flyCounter = fly(
  ({ getFly }) => {
    // AppComponent が指定されてないのでエラーになる
    const {/*...*/} = getFly(flyAny);

    /*...*/
  },  
  [ AppComponent ] 
)
```


### メモ化を考えてみる

次はメモ化について考えていきます。
とは言っても、`fly()` を使って実装できるので State とそれほど変わらないです👇

```tsx:couterの値を使ってdisplayTextをメモ化してみる
import { fly } from "xxx"

export const counterState = fly(
  ({ memo }) => {
    const [counter, setCounter] = createState(0);

    // counter の値が変わると displayText の値が変わる
    const displayText = memo(() => `${counter} times`, [counter])

    return { displayText, setCounter }

  },  
  [ AppComponent ] 
)
```

上記の例では、`counter` を使って `displayText` をメモ化しています。( メモ化するほどの処理じゃないし、例としては良くないコードだけど本題じゃないから許してッピ🐙 )

基本的な挙動は useMemo と大して変わらない感じで書けるので、useMemo を使うような感覚で使えるはずです。

また、関数をメモ化したい場合は `memoCallback` を使用します👇

```tsx:関数をメモ化してみる
import { fly } from "xxx"

export const counterState = fly(
  ({ memoCallback }) => {
    const [counter, setCounter] = createState(0);

    // 引数に渡した関数が返されることに注意
    const handleClick = memoCallback(() => {
      setCounter(counter + 1);      
    }, [counter])

    return { handleClick }
  },  
  [ AppComponent ] 
)

 /*...*/
```

こちらも useCallback とほとんど変わらない書き方で実装できます。


### ref オブジェクトを考えてみる

次は Ref オブジェクトを扱う方法を考えます。
Ref オブジェクトは React 管理外の処理に使われるのでどうするか迷いましたが、State の時と同様に `fly()` 内で定義できるようにします👇

```tsx:Refオブジェクトを実装してみる
import { fly } from "xxx"

export const flyCounterRef = fly(
  ({ ref }) => {
    const counter = ref<number>(0);
    const setCounter = (value: number) => {
      counter.current = value;
    }

    return { counter, setCounter }
  },  
  [ AppComponent ] 
)
```

注意点として、useRef と同じように実際の値にアクセスするには、`.current` からアクセスする必要があります。

また、DOM を取得するようなこともできます👇

```tsx:JSXにRefを設定できるようにしてみる
import { fly, type GetFlyType } from "xxx"

const flyRefs = fly(
  ({ ref }) => {
    const divRef = ref<HTMLDivElement | null>(null);
    return { divRef }
  },  
  [ AppComponent ] 
)

interface AppProps {
  flyRefs: GetFlyType<typeof flyRefs>
}

export function AppComponent({ flyRefs }: AppProps) {
  return (
    <div ref={flyRefs.devRef}>{/*...*/}</div>
  )
}
```

ただし、このままだと DOM が描画された時の処理ができないため、実践では後述する `flyDidMount()` と併用する形になると思います。


### マウント時のイベント処理を考えてみる

次は React Hooks で言うところの useEffect などのマウント時の処理について考えますが、こちらも `fly()` の時と実装はあんまり変わりません👇

```tsx:マウント時の処理を実装してみる
import { AppComponent } from "./AppComponent.tsx"
import { fly, flyDidMount } from "xxx"

export const flyAny = fly(/*...*/, [ AppComponent ])

// 第二引数に指定したコンポーネントがマウント(更新される)たびに関数が実行されます
export const flyAnyEffect = flyDidMount(
  ({ getFly }) => {
    // State や Ref を取得する
    const { anyState, anyRef } = getFly(flyAny)

    // アンマウントされる時に実行される関数
    const willUnmount = () => {/*...*/}
    return willUnmount;
  },
  [ AppComponent ]
)
```

ほとんど useEffect と似たように書けますが、マウント(更新)されるたびに必ず実行される点には注意する必要があります。

また、初回マウント時のみ処理したい場合は `flyDidMountOnce` という別の関数を使用する必要があります👇

```tsx:初回マウント時の処理を実装してみる
import { AppComponent } from "./AppComponent.tsx"
import { fly, flyDidMountOnce } from "xxx"

const flyAny = fly(/*...*/, [ AppComponent ])

// flyDidMount() と同じように実装できます
export const flyAnyEffect = flyDidMountOnce(
  ({ getFly }) => {
    // Stateを取得する
    const state = getFly(flyAny)

    // アンマウントされる時に実行される関数
    const willUnmount = () => {/*...*/}
    return willUnmount;
  }
  [ AppComponent ]
)
```

### グローバルステートを考えてみる

通常の `fly()` はローカルな処理しか実装できないので、少し不便だと設計の時点で感じました。せっかくコンポーネントの外に書けるならグローバルな処理もやりたい！ってことで、グローバルな処理も考えてみます👇

```tsx:グローバルな処理の実装
import { AppComponent } from "./AppComponent.tsx"
import { flyGlobal } from "xxx"

/**
 * fly() と実装は同じですが、初期化は一回しかされず、
 * 全てのコンポーネントで同じ State が共有されます
 */
export const flyGlobalCounter = flyGlobal(
  ({ createState }) => {
    const [counter, setCounter] = createState(0)
    return { counter, setCounter }
  },
  [ AppComponent ]
)
```

実装は `fly()` の時とほとんど変わりませんがが、グローバルなのでStateは全てのコンポーネントで共有される点には注意する必要があります。しかし、使用するコンポーネントは第二引数の配列に指定する必要があるため、影響範囲を最小限に抑えられると思っています。

また `flyGlobal()` でも依存するコンポーネントが把握できるので、より効率の良い更新を実現できそうなのも、筆者的にポイント高いです。

### 非同期処理も考えてみる

次は非同期処理について考えてみます。
React では Suspense などがありますが、Promise を throw する関係上、 JSX の構築が難しくなったり、Error 以外の値を throw するという思想的な問題もはらんでいる印象です。

fly では一連の API をコンポーネントの外に出したおかげで無理なく非同期処理を組み込めると思っています。実装は以下のような感じ👇

```tsx
import { flyAsnyc, GetFlyAsyncType } from "xxx"

type FetchData = any;

export const flyFetchData = flyAsnyc(
  async ({ createState }) => {
    const initialValue = await fetch("...");
    const [data, setData] = createState<FetchData>(initialValue)

    return { data, setData }
  },
  [ AppComponent ]
)

interface AppProps {
  flyFetchData: GetFlyAsyncType<typeof flyFetchData>
  // 👆 の型は `{ data: ..., setData: ... } | null` になります
}

export function AppComponent({ flyFetchData }: AppProps) {
  if(flyFetchData === null) {
    return <p>Loading...</p>
  }

  const { data, setData } = flyFetchData;
  
  return /* ... */
}
```

非同期処理なため、どうしても Props に渡される値は全て Nullable になります。そのため、処理が完了して無い場合の処理を実装する必要があります👇

```tsx:非同期処理が終わってない場合の処理をコンポーネントに書く必要があります
export function AppComponent({ flyFetchData }: AppProps) {
  if(flyFetchData === null) {
    return <p>Loading...</p>
  }
  /*...*/
}
```

これには賛否両論あると思いますが、コンポーネントをテストしやすくなるメリットなどがあると思うので、この設計にしました。( 他のライブラリとかもこの方式を使っていたりするし大丈夫な気がしますが )


## fly のメリットとデメリットを考えてみる

はい。ここまで fly の API について考えてきましたが、そこから一歩踏み込んで「もし実装されたら？」という仮定で、この API のメリット・デメリットを~~まだ作っても無いのに~~考えていきたいと思います。

まずは**メリット**から、

- fly を使うとコンポーネントの参照透過性を高く保ちやすい
- コンポーネントに依存せずに書ける(JSXと分けて書ける)
- パフォーマンスを高くできる(かも？)
- 依存範囲を把握しやすい
- ツール群が解析しやすい

とりあえずこんなところでしょうか。

個人的に `fly を使うとコンポーネントの参照透過性を高く保ちやすい` は一番のメリットかなと思います。元々、関数型コンポーネントのメリットの一つに参照透過性があるというのがあったんですが、React Hooks 登場によりそれも無くなってしまったので、fly によって関数型コンポーネントの魅力をより引き出せるのではないかと考えています。

また依存範囲を把握しやすい書き方をするので、もし ESLint などの構文解析して何かするツールを作ろうとした時でも、比較的簡単に実装しやすいのではないかと思っています。(実際に作ってないので本当のところは分かりませんが...)


次は**デメリット**について考えます。

- 記述量が多くなる
- コンポーネントと分離して実装するので、実装しにくい部分はあるかも
- React の思想と合ってない部分もあるかも
- React Hooks とは違った処理フローになれる必要がある

やっぱり、コンポーネントの外に書く必要があるので、コンポーネントを実装しながら State を実装するというのが難しいですし、必然的に記述量も多くなる傾向にあると思います。なので、慣れないうちは記述量の多さも相まってデメリットしか感じないかもしれません。

また、React Hooks と一緒に使うと fly のメリットが半減したり、それぞれの処理フローを把握する必要があるので、開発者の負担を大きくしてしまうかもしれません。


## あとがき

っとまぁ、ここまで在りもしない API について考えてきましたが、ぶっちゃけ実際に使ってないので、ここで色々と考えても鬼が笑うだけっていうね😇

でも、自分では「 良さそう！ 」って思っているので、React を使った新しいフレームワークを作ってみるのも勉強になるしアリかな～と思っています。~~思っているだけ~~

一応、[packelyze](https://github.com/mizchi/packelyze) や [hono](https://hono.dev/) などを使えば各種 Edge ランタイムをターゲットとしたフレームワークは作れそうなので、「どの Edge ランタイムでも動く Next.js！」みたいな思想のフレームワークは作ってみてもいいかも？~~知らんけど~~

ここまでくだらない記事を読んで頂いてありがとうございます🙏

これが誰かの参考になれば幸いです。
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。

それではまた 👋