---
title: "バリバリ軽量No.1な検証ライブラリ Valibot の紹介"
emoji: "👹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["valibot"]
published: true
---


## この記事について

この世はわからない　値がたくさんある　どんな値が来ても　負けない処理を書こう
それでも脆弱性　必ずあるもんだ　守ってあげましょう　それが強さなんだ
今日から一番　たくましいのだ　お待たせしました　凄い奴
今日から一番　カッコイイのだ　バリバリ軽量No.1なライブラリ Valibot の紹介記事です。

なるほど～ってなったところで、ホント今日から 一番 一番だ 一番!!を目指して解説していこうかと思います🌸


## Valibot について

https://valibot.dev/

JavaScript / TypeScript で使える検証ライブラリです。
特徴としては、

- 強力な型推論による型生成
- 600 バイト未満の小さなバンドル サイズ
- 多くの変換および検証処理が含まれています
- 依存関係がゼロ
- 最小限で読みやすいAPI

などあります。

個人的には `最小限で読みやすいAPI` がとても良い点だと思っていて、公式ドキュメント見ると以下のような構成になっています 👇

![](https://storage.googleapis.com/zenn-user-upload/a9d963c25583-20240627.png)
*https://valibot.dev/guides/mental-model/ より引用*

schema で ”型” を検証し、action で ”値” を処理し、それらを pipe でまとめる。と言った構造になっていて、Zod などのメソッドチェーンで記述する方法より記述量は多くなってしまうんですが、それぞれの役割が明確に分離されていて、とても分かりやすい API になってます。

最近のフロントエンドは関数型っぽい記述が増えているので、valibot の API 設計の方がメソッドチェーンで記述するよりも今のフロントエンドにあっているのではないかな～と個人的に思っています。


あと、そろそろ？メジャーリリースが発表されるそうなので、今のうちに触っておくのもアリよりのアリだと思います。

https://github.com/fabian-hiller/valibot/releases/tag/v1.0.0-beta.1


## 実行環境について

記事内の実装は以下のバージョンで行っています。

```json:valibot のバージョン
"dependencies": {
  "valibot": "0.42.1"
}
```


## 基本的な使い方

検証したい型のスキーマを作り、そのスキーマを使って検証します。


```ts:parse()を使った例
import * as v from "valibot";

const StringSchema = v.string(); // スキーマを生成

const inputValue: unknown; // 入力値を定義

try {
  // ここで検証するがバリデーションに引っかかると throw するので注意！
  const result: string = v.parse(StringSchema, inputValue);
  console.log(`${result.output} is string`);
} catch (e: unknown) {
  // valibot のエラーか判定します 
  if (v.isValiError(e)) console.error(e);
}
```

### throw させないで検証する

上記のように `v.parse()` だと検証に引っかかった時に throw するので、throw させたくない場合は `v.safeParse()` を使用します。

```ts:safeParse()を使った例
import * as v from "valibot";

const StringSchema = v.string(); // スキーマを生成

const inputValue: unknown; // 入力値を定義

const result = v.safeParse(StringSchema, inputValue); // 検証する

if (result.success) console.log(`${result.output} is string`);
else console.error(new v.ValiError(result.issues));
```

注意点として、`safeParse()` の返り値には `ValiError` が無いため、もし `ValiError` が欲しいなら `issues` の値から値を作成する必要があります 👇

```ts:ValiErrorを作成する
const result = v.safeParse(StringSchema, inputValue); // 検証する
const error = new v.ValiError(result.issues); // エラーオブジェクトを作成する
```

### 型の生成

定義したスキーマから型を生成できます。

```ts:型の生成
import * as v from "valibot";

const Schema = v.object({
  name: v.string(),
  age: v.number(),
})

type UserType = v.InferOutput<typeof Schema>; // { name: string; age: number }
```

また、`v.transform()` などで入力値と出力値の型が違う場合は、`v.InferInput<T>` を使用することで入力値側の型を取得することができます。

```ts:InferInputを使って入力値の型も生成できる
import * as v from 'valibot';

const Schema = v.object({
  date: v.pipe(v.string(), v.transform(str => new Date(str)))
});

type InputType = v.InferInput<typeof Schema> // { date: string; }
type OutputType = v.InferOutput<typeof Schema> // { date: Date; }
```


### エラーメッセージの指定とエラーオブジェクト(issues)について

スキーマを定義する時にエラーメッセージを設定できます。

```ts
import * as v from "valibot";

const UserSchema = v.object({
  name: v.string(),
  age: v.number(),
}, "入力値は User 型ではありませんでした")

try {
  v.parse(UserSchema, value);
} catch (e) {
  console.log(e.message); // → "入力値は User 型ではありませんでした"
}
```

また、`v.parse()` などで検証に失敗すると、`ValiError` という Error クラスを継承したインスタンスが返されます。このインスタンスには `issues` という検証に関する情報が含まれているので、その値を使って検証失敗後の処理を簡単に扱えるようになります 👇

```ts
import * as v from 'valibot';

const Schema = v.pipe(v.string(), v.minLength(8));

try {
  const result = v.parse(Schema, "");
} catch (err) {
  if(!v.isValiError(err)) throw err;

  const issue = err.issues[0]
  console.log(`文字数は${issue.requirement}文字以上にする必要があります`)
  // → 文字数は8文字以上にする必要があります
}
```


#### まとめ

はい。以上で基本的な使い方は終了です。
次からはスキーマの定義方法について見ていきましょう⚗


## string ( 文字列型 )

```ts
import * as v from "valibot";

v.string();                            // 単純な文字列
v.pipe(v.string(), v.length(5))        // 固定幅 ( length === 5 )
v.pipe(v.string(), v.minLength(5))     // 5文字以上 ( length >= 5 )
v.pipe(v.string(), v.maxLength(5))     // 5文字以下 ( length <= 5 )
v.pipe(v.string(), v.notLength(5))     // 5文字以外 ( length !== 5 )
v.pipe(v.string(), v.email())          // メールアドレス
v.pipe(v.string(), v.regex(RegExp))    // 引数に渡した正規表現にマッチする文字列
v.pipe(v.string(), v.url())            // URL
v.pipe(v.string(), v.emoji())          // Emoji
v.pipe(v.string(), v.uuid())           // UUID 
v.pipe(v.string(), v.cuid2())          // Cuid2
v.pipe(v.string(), v.ulid())           // ULID
v.pipe(v.string(), v.bic())            // BIC ( SWIFTコード )
v.pipe(v.string(), v.bytes(5))         // 固定バイト
v.pipe(v.string(), v.minBytes(5))      // 5 バイト以上
v.pipe(v.string(), v.maxBytes(5))      // 5 バイト以下
v.pipe(v.string(), v.notBytes(5))      // 5 バイト以外
v.pipe(v.string(), v.creditCard())     // クレジットカード番号
v.pipe(v.string(), v.octal())          // 8進数の文字列
v.pipe(v.string(), v.decimal())        // 10進数の文字列
v.pipe(v.string(), v.hexadecimal())    // 16進数の文字列
v.pipe(v.string(), v.startsWith("@"))  // 最初が"@"の文字列
v.pipe(v.string(), v.endsWith("@"))    // 最後が"@"の文字列
v.pipe(v.string(), v.includes("@"))    // "@"を含む文字列
v.pipe(v.string(), v.excludes("@"))    // "@"を含まない文字列
v.pipe(v.string(), v.hash(["md5"]))    // "md5"のハッシュ文字列
v.pipe(v.string(), v.hexColor())       // 16進数カラーコード文字列
v.pipe(v.string(), v.imei())           // IMEI 文字列 ( 端末識別番号 )
v.pipe(v.string(), v.ip())             // IPv4 または IPv6 文字列
v.pipe(v.string(), v.ipv4())           // IPv4 文字列
v.pipe(v.string(), v.ipv6())           // IPv6 文字列
v.pipe(v.string(), v.isoDate())        // "YYYY-MM-DD" のような文字列
v.pipe(v.string(), v.isoDateTime())    // "2023-06-31T00:00" のような文字列
v.pipe(v.string(), v.isoTime())        // "hh:mm" のような文字列
v.pipe(v.string(), v.isoTimeSecond())  // "hh:mm:ss" のような文字列
v.pipe(v.string(), v.isoTimestamp())   // "2023-06-31T00:00:00.000Z" のような文字列
v.pipe(v.string(), v.isoWeek())        // "2023-W14" のような文字列
v.pipe(v.string(), v.mac())            // 48-bit または 64-bit MAC アドレス
v.pipe(v.string(), v.mac48())          // 48-bit MAC アドレス
v.pipe(v.string(), v.mac64())          // 64-bit MAC アドレス
```

#### email() について

メールアドレスは以下の正規表現を使って検証されている 👇

```ts
/**
 * Email regex.
 */
export const EMAIL_REGEX =
  /^[\w+-]+(?:\.[\w+-]+)*@[\da-z]+(?:[.-][\da-z]+)*\.[a-z]{2,}$/iu;
```
>引用: https://github.com/fabian-hiller/valibot/blob/7fa24c480576d5ed59680c56281ce1e2efc29149/library/src/regex.ts#L30-L31

#### url() について

`new URL()` でパースできる値。どのような値がパースできるかは以下のドキュメント参照👇

https://developer.mozilla.org/ja/docs/Web/API/URL/URL

#### bytes() について

文字列のバイト数を検証します。
例えば `"a"` なら 1 バイトですが `"あ"` は 3 バイトになるので、その辺りを考慮した検証に使います。バイト数を数える実装は以下の通りです 👇

```ts
const length = new TextEncoder().encode(input).length;
```

>引用: https://github.com/fabian-hiller/valibot/blob/7fa24c480576d5ed59680c56281ce1e2efc29149/library/src/utils/_getByteCount/_getByteCount.ts#L12-L17

#### creditCard() について

サポートしているカード番号は `American Express`, `Diners Card`, `Discover`, `JCB`, `Union Pay`, `Master Card`, `Visa`。

注意として、クレジットカード番号は単純な正規表現を使って検証されているので、使用可能な番号か判定するには別途実装が必要。( ハイフンは含まれていても OK 👌 )

```ts
const PROVIDER_REGEX_LIST = [
  // American Express
  /^3[47]\d{13}$/u,
  // Diners Club
  /^3(?:0[0-5]|[68]\d)\d{11,13}$/u,
  // Discover
  /^6(?:011|5\d{2})\d{12,15}$/u,
  // JCB
  /^(?:2131|1800|35\d{3})\d{11}$/u,
  // Mastercard
  /^5[1-5]\d{2}|(?:222\d|22[3-9]\d|2[3-6]\d{2}|27[01]\d|2720)\d{12}$/u,
  // UnionPay
  /^(?:6[27]\d{14,17}|81\d{14,17})$/u,
  // Visa
  /^4\d{12}(?:\d{3,6})?$/u,
];
```

>引用: https://github.com/fabian-hiller/valibot/blob/7fa24c480576d5ed59680c56281ce1e2efc29149/library/src/actions/creditCard/creditCard.ts#L78-L93

クレジットカード番号の仕様については以下の wiki を参照とのこと 👇

https://ja.wikipedia.org/wiki/クレジットカードの番号

#### hash() について

検証できるハッシュ文字列は以下の通り👇

```ts
type HashType = 
  | 'md4'
  | 'md5'
  | 'sha1'
  | 'sha256'
  | 'sha384'
  | 'sha512'
  | 'ripemd128'
  | 'ripemd160'
  | 'tiger128'
  | 'tiger160'
  | 'tiger192'
  | 'crc32'
  | 'crc32b'
  | 'adler32'
```
> 引用: https://valibot.dev/api/HashType/


## number ( 数値型 )

```ts
import * as v from "valibot"

v.number()                          // 単純な数値型
v.pipe(v.number(), v.finite())      // 有限な値 ( Number.isFinite(n) )
v.pipe(v.number(), v.integer())     // 整数 ( Number.isInteger(n) )
v.pipe(v.number(), v.minValue(5))   // 5 以上 ( n >= 5 )
v.pipe(v.number(), v.maxValue(5))   // 5 以下 ( n <= 5 )
v.pipe(v.number(), v.multipleOf(5)) // 5 の倍数 ( n % 5 === 0 )
v.pipe(v.number(), v.notValue(5))   // 5 ではない ( n !== 5 )
v.pipe(v.number(), v.safeInteger()) // 安全な整数 ( Number.isSafeInteger(n) )
v.pipe(v.number(), v.toMinValue(5)) // 5 以下の値を 5 にする ( Math.min(n, 5) )
v.pipe(v.number(), v.toMaxValue(5)) // 5 以上の値を 5 にする ( Math.max(n, 5) )
v.pipe(v.number(), v.value(5))      // 5 という数 ( n === 5 )
```

:::message
`v.number()` は [NaN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/NaN) や [BigInt](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/BigInt) を受け付けないので注意してください。
:::

#### v.finite() について

有限な値であることを検証しますが、検証には[Number.isFinite()]()を使用しています。
そのため、この検証に失敗する値は

- 正の無限大 (Infinity)
- 負の無限大 (Infinity)

になります。

参考: 

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Number/isFinite

#### v.safeInteger() について

安全な整数であることを検証しますが、検証には [Number.isSafeInteger()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Number/isSafeInteger) を使用しています。

安全な整数については、以下のドキュメントが参考になります👇

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Number/isSafeInteger#解説

### NaN

[NaN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/NaN) を判定するには、`v.nan()` と言う別のスキーマで判定する必要があります。

```ts
import * as v from "valibot";

v.nan() // NaN か判定するスキーマ
```

### bigint

[BigInt](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/BigInt) を判定するには、`v.bigint()` と言う別のスキーマで判定する必要があります。


```ts
import * as v from 'valibot';

v.bigint()                           // 単純な数値型
v.pipe(v.bigint(), v.minValue(5n))   // 5n 以上 ( n >= 5n )
v.pipe(v.bigint(), v.maxValue(5n))   // 5n 以下 ( n <= 5n )
v.pipe(v.bigint(), v.notValue(5n))   // 5n ではない ( n !== 5n )
v.pipe(v.bigint(), v.toMinValue(5n)) // 5n 以下の値を 5n にする
v.pipe(v.bigint(), v.toMaxValue(5n)) // 5n 以上の値を 5n にする 
v.pipe(v.bigint(), v.value(5n))      // 5n という数 ( n === 5n )
```


## boolean ( 論理型 )

```ts
import * as v from "valibot";

v.boolean() // 論理値のスキーマ
```

:::message
`v.boolean()` にも [v.value()](https://valibot.dev/api/value/) などのバリデーションを設定できますが、取る値が `true` か `false` しかないので、もしどちらかを一方を検証したいなら [`v.literal()`](#comment-4845c6bbce0408) を使用した方が良いと思います。
:::


## date ( 日付型 )

```ts
import * as v from "valibot";

const date = new Date(2019, 0, 1)

v.date() // Date オブジェクト
v.pipe(v.date(), v.minValue(date)) // 2019/01/01 より後の日付 (n > date)
v.pipe(v.date(), v.maxValue(date)) // 2019/01/01 より前の日付 (n < date)
v.pipe(v.date(), v.value(date)     // 2019/01/01 の日付オブジェクト

// 範囲を設定する ( 2019/01/01 ~ 2020/01/01 までの日付 )
v.pipe(
  v.date(),
  v.minValue(new Date(2019, 0, 1))
  v.maxValue(new Date(2020, 0, 1))
)
```


## literal ( リテラル型 )

```ts
import * as v from "valibot";

v.literal("foo")          // 文字列リテラル
v.literal(0)              // 数値リテラル
v.literal(true)           // 論理値リテラル
v.literal(Symbol("hoge")) // Symbolリテラル
v.literal(BigInt(1))      // BigIntリテラル
v.literal(null)           // nullリテラル
v.literal(undefined)      // undefinedリテラル
```

:::message
設定できる値は、`null | undefined | number | string | boolean | symbol | bigint` となっています。
:::


## null

```ts
import * as v from "valibot";

v.null_() // null か判定するためのスキーマ
```

### nullable と nonNullable について

入力値に `null` を許容するスキーマを作成する時に使います。

```ts
import * as v from "valibot";

v.nullable(v.string()); // `string | null` のスキーマ
```

デフォルト値を設定するには以下のようにします。

```ts:デフォルト値を設定する
import * as v from "valibot";

// デフォルト値を設定
const schema = v.nullable(v.string(), () => "default value");

// 関数を使用することもできる
v.nullable(v.string(), () => "default value");

// 型を出力すると `string | null` ではなく `string` になることに注意！
type SchemaType = v.InferOutput<typeof schema>

v.parse(schema, "hoge") // → "hoge"
v.parse(schema, null)   // → "default value"
```

また、元に戻すには `nonNullable` を使用します 👇

```ts
import * as v from "valibot";

const schema = v.nullable(v.string()); // `string | null` のスキーマ
v.nonNullable(schema); // `string` のスキーマに変換
```

### nullish と nonNullish について

入力値に `null` と `undefined` を許容するスキーマを作成する時に使います。

```ts
import * as v from "valibot";

v.nullish(v.string()); // `string | null | undefined` のスキーマ
```

デフォルト値を設定するには以下のようにします。

```ts:デフォルト値を設定する
import * as v from "valibot";

// デフォルト値を設定
const schema = v.nullish(v.string(),  "default value");

// 関数を使用することもできる
v.nullish(v.string(), () => "default value");

// 型を出力すると `string | null | undefined` ではなく `string` になることに注意！
type SchemaType = v.InferOutput<typeof schema>

v.parse(schema, "hoge") // → "hoge"
v.parse(schema, null)   // → "default value"
```

また、元に戻すには `nonNullable` を使用します 👇

```ts
import * as v from "valibot";

const schema = v.nullish(v.string()); // `string | null | undefined` のスキーマ
v.nonNullish(schema); // `string` のスキーマに変換
```

## undefined 

```ts
import * as v from "valibot";

v.undefined_() // undefined か判定するためのスキーマ
```

### optional と nonOptional について

入力値に `undefined` を許容するスキーマを作成する時に使用します。

```ts
v.optional(v.string()) // `strnig | undefined` のスキーマ
```

デフォルト値を設定するには以下のようにします。

```ts:デフォルト値を設定する
import * as v from "valibot";

// デフォルト値を設定
const schema = v.optional(v.string(), "default value");

// 関数を使用することもできる
// v.optional(v.string(), () => "default value");

// 型を出力すると `string | undefined` ではなく `string` になることに注意！
type SchemaType = v.InferOutput<typeof schema>

v.parse(schema, "hoge") // → "hoge"
v.parse(schema, null)   // → "default value"
```

また、元に戻すには `nonOptional` を使用します👇

```ts
import * as v from "valibot";

const schema = v.optional(v.string()); // `string  | undefined` のスキーマ
v.nonOptional(schema); // `string` のスキーマに変換
```


## any / unknown / never / void

主に TypeScript の型情報を付与するためのスキーマです。


```ts
v.any()     // どんな値にもマッチするスキーマ
v.unknown() // どんな値にもマッチするスキーマ
v.never()   // どんな値にもマッチしないスキーマ
v.void_()   // undefined か判定するスキーマ
```

`v.void()` については、`v.undefined()` と一緒ですが、型情報が void になります。undefined と void の違いについては以下のサイトが分かりやすかったです👇

https://typescriptbook.jp/reference/functions/void-type


## array ( 配列型 )

```ts
import * as v from "valibot";

v.array(v.string())            // string のみを含む配列

const arr = v.array(v.string())

v.pipe(arr, v.minLength(5))    // 要素数が 5 以上の配列 ( n >= 5 )
v.pipe(arr, v.maxLength(5))    // 要素数が 5 以下の配列 ( n <= 5 )
v.pipe(arr, v.includes('foo')) // 'foo' を含む配列
v.pipe(arr, v.excludes('foo')) // 'foo' を含まない配列
v.pipe(arr, v.empty())         // 空配列
v.pipe(arr, v.nonEmpty())      // 空配列ではない配列
v.pipe(arr, v.readonly())      // readonlyな配列
v.pipe(arr, v.every(fn))       // fn が判定した要素が全て true を返した配列 
v.pipe(arr, v.some(fn))        // fn が判定した要素のいずれかが true を返した配列
```

#### v.readonly() について

型情報を付与するだけで実際の検証には何も処理しません。以下は valibot のソースコードの引用です👇

```ts
export function readonly<TInput>(): ReadonlyAction<TInput> {
  return {
    kind: 'transformation',
    type: 'readonly',
    reference: readonly,
    async: false,
    '~validate'(dataset) {
      return dataset;
    },
  };
}
```

>引用: https://github.com/fabian-hiller/valibot/blob/7fa24c480576d5ed59680c56281ce1e2efc29149/library/src/actions/readonly/readonly.ts#L23-L33


## map

```ts
import * as v from "valibot";

v.map(v.string(), v.number()) // Map<string, number> か判定するためのスキーマ

const schema = v.map(v.string(), v.number())
v.pipe(schema, v.minSize(5)) // 要素数が 5 以上の Map ( size >= 5 )
v.pipe(schema, v.maxSize(5)) // 要素数が 5 以下の Map ( size <= 5 )
v.pipe(schema, v.size(5))    // 要素数 5 の Map ( size === 5 )
```


## set

```ts
import * as v from "valibot";

v.set(v.string()) // Set<string> か判定するためのスキーマ

const schema = v.set(v.string())
v.pipe(schema, v.minSize(5)) // 要素数が 5 以上の Set ( size >= 5 )
v.pipe(schema, v.maxSize(5)) // 要素数が 5 以下の Set ( size <= 5 )
v.pipe(schema, v.size(5))    // 要素数 5 の Set ( size === 5 )
```


## object ( オブジェクト型 )

```ts
import * as v from "valibot";

// { key1: string; key2: number; } のスキーマ
v.object({
  key1: v.string(),
  key2: v.number(),
});
```

:::message
`v.object()` では定義されてない要素については検証せず、また出力結果にも含めないことに注意してください。以下の例では検証には成功しますが、定義されていない `key3: "hoge"` が削除されています 👇

```ts:v.object()は定義されてない要素を含めないことに注意！
import * as v from "valibot";

const schema = v.object({
  key1: v.string(),
  key2: v.number(),
});

const input = { key1: "hello", key2: 555, key3: "hoge" };
const result = v.parse(schema, input)

console.log(input)  // → { key1: "hello", key2: 555, key3: "hoge" }
console.log(result) // → { key1: "hello", key2: 555 }
```

この挙動を変えたい場合は `v.looseObject()`, `v.strictObject()`, `v.objectWithRest()` を使用してください。
:::


### looseObject について

`v.object()` と似ていますが、検証結果に定義していない要素を含む点が違います。

```ts
import * as v from "valibot";

const Schema = v.looseObject({
  key1: v.string(),
  key2: v.number(),
});

const input = { key1: "hello", key2: 555, key3: "hoge" };
const result = v.parse(Schema, input)

console.log(input)  // 👉 { key1: "hello", key2: 555, key3: "hoge" }
console.log(result) // 👉 { key1: "hello", key2: 555, key3: "hoge" }
```

:::message
検証結果に定義してない要素が含まれていますが、その要素は検証されてない点に注意してください。
:::

### strictObject について

`v.object()` では定義されてない要素を無視しましたが、こちらのスキーマは厳格にチェックします。

```ts
import * as v from "valibot";

const schema = v.strictObject({
  key1: v.string(),
  key2: v.number(),
});

const input = { key1: "hello", key2: 555, key3: "hoge" };

try {
  const result = v.parse(schema, input)
} catch (e) {
  console.error(e)
  // 👉 ValiError: Invalid type: Expected never but received "hoge"
}
```

### objectWithRest について

`v.object()` のように定義してない要素を値無視するのではなく、別のスキーマで検証したい場合はこちらを使います。

```ts
import * as v from "valibot";

// 定義してない要素が null であることを検証するスキーマ
const schema = v.objectWithRest(
  {
    key1: v.string(),
    key2: v.number(),
  },
  v.null_()
);

// 定義してない`key3`が null なので検証に成功する
const validInput = { key1: "", key2: 0, key3: null }
const validResult = v.safeParse(schema, validInput)
console.log(validResult) // 👉 { success: true, ... }

// 定義してない`key3`が undefined なので検証に失敗します
const invalidInput = { key1: "", key2: 0, key3: undefined }
const invalidResult = v.safeParse(schema, invalidInput)
console.log(invalidResult) // 👉 { success: false, ... }
```


## tuple

```ts
import * as v from "valibot";

v.tuple([v.string(), v.number()]) // [string, number] のスキーマ
```

:::message
`v.tuple()` では定義されてない要素については検証せず、また出力結果にも含めないことに注意してください。

以下の例では検証には成功しますが定義されていない `null` が削除されています 👇

```ts:v.tuple()は定義されてない要素を無視する点に注意！
import * as v from "valibot";

const schema = v.tuple([v.string(), v.number()]) 

const input = ["hello!", 0, null]
const result = v.parse(schema, input)

console.log(input)  // 👉 ["hello!", 0, null]
console.log(result) // 👉 ["hello!", 0]
```

この挙動を変えたい場合は `v.looseTuple()`, `v.strictTuple()`, `v.tupleWithRest()` を使用してください。
:::

### looseTuple について

`v.tuple()` とほとんど同じですが、検証結果に定義してない要素も含める点が違います。

```ts
import * as v from "valibot";

const schema = v.looseTuple([v.string(), v.number()]) 

const input = ["hello!", 0, null]
const result = v.parse(schema, input)

console.log(input)  // 👉 ["hello!", 0, null]
console.log(result) // 👉 ["hello!", 0, null]
```

:::message
検証結果に定義してない要素が含まれていますが、その要素は検証されてない点に注意してください。
:::

### strictTuple について

`v.tuple()` では定義されてない要素を無視しましたが、こちらのスキーマは厳格にチェックします。

```ts
import * as v from "valibot";

const schema = v.strictTuple([v.string(), v.number()]);

const input = ["", 0, null]

try {
  v.parse(schema, input);
} catch (e) {
  console.error(e); 
  // 👉 ValiError: Invalid type: Expected never but received null
}
```

### tupleWithRest について

`v.tuple()` のように定義した要素以外の値を無視するのではなく、別のスキーマで検証したい場合はこちらを使います。

```ts
import * as v from "valibot";

// 2番目以降の要素がnullであることを検証するスキーマ
const schema = v.tupleWithRest([v.string(), v.number()], v.null_());

const validResult = v.safeParse(schema, ["", 0, null])
console.log(validResult) // 👉 { success: true, ... }

const invalidResult = v.safeParse(schema, ["", 0, undefined])
console.log(invalidResult) // 👉 { success: false, ... }
```

また、型情報は以下のようになります👇

```ts:tupleWithRestから出力した型
type SchemaType = v.InferOutput<typeof schema>
// 👉 [string, number, ...null[]]
```


## record

[Record](https://www.typescriptlang.org/docs/handbook/utility-types.html#recordkeys-type) 型のスキーマを作成する時に使用します。

```ts
import * as v from 'valibot';

v.record(v.string(), v.number()) // Record<string, number> のスキーマ
```


## intersect ( 交差型 )

交差型のスキーマを作成するときに使用します。

```ts
import * as v from 'valibot';

const schema = v.intersect([
  v.object({ foo: v.string() }),
  v.object({ bar: v.number() }),
]);

type SchemaType = v.InferOutput<typeof schema>
// { foo: string } & { bar: number }
```

:::message
`v.intersect()` は検証時に渡されたスキーマをそれぞれ実行することに注意してください！
:::


### 複数の `v.object()` をマージしたい

`v.intersect()` とは違い複数の `v.object()` から一つの `v.object()` を生成した場合は以下のようにします。

```ts
const fooSchema = v.object({ foo: v.string(), hoge: v.boolean() });
const barSchema = v.object({ bar: v.string(), hoge: v.string() });

const MergedSchema = v.object({
  ...fooSchema.entries,
  ...barSchema.entries,
});

type SchemaType = v.InferOutput<typeof schema>
// { foo: string; bar: string; hoge: string }
```

:::message
重複しているプロパティがあった場合は最後に定義されたプロパティが使用されます。上記の例では `barSchema` の `hoge: v.string()` が優先されています。
:::


## union ( 直和型 )

```ts
import * as v from "valibot";

v.union([v.string(), v.number()]) // `string | number` のスキーマ
```

### variant ( 判別可能な union ) について

 [判別可能なユニオン型 (discriminated union)](https://typescriptbook.jp/reference/values-types-variables/discriminated-union) を定義するためには `v.variant()` を使用します。

```ts
import * as v from "valibot";

// 判別可能なユニオン型 のスキーマ
v.variant("type", [
  v.object({
    type: v.literal("foo"),
    foo: v.string(),
  }),
  v.object({
    type: v.literal("bar"),
    bar: v.number(),
  }),
]);
```


## enum ( 列挙型 )

```ts
import * as v from "valibot";

const MyEnum = { Foo: "FOO", Bar: "BAR" } as const;
v.enum_(MyEnum) // "FOO" | "BAR" の列挙型のスキーマを定義

// TypeScript の enum を使用する事もできます
enum MyTsEnum {
  Foo = "FOO",
  Bar = "BAR",
}
v.enum_(MyTsEnum) // "FOO" | "BAR" の列挙型のスキーマを定義
```

### picklist ( 配列による列挙型 )について

配列から列挙型を定義したい場合は `v.picklist()` を使用します。

```ts
import * as v from 'valibot';

v.picklist(['FOO', 'BAR'] as const) // "FOO" | "BAR" の列挙型のスキーマ
```


## blob

```ts
import * as v from "valibot";

v.blob()                                      // Blobオブジェクトか判定するスキーマ
v.pipe(v.blob(), v.minSize(1024 * 1024 * 10)) // 10MB以上のBlob ( size >= 10MB )
v.pipe(v.blob(), v.maxSize(1024 * 1024 * 10)) // 10MB以下のBlob ( size <= 10MB )
v.pipe(v.blob(), v.size(1024 * 1024 * 10))    // 10MBのBlob ( size == 10MB )
v.pipe(v.blob(), v.mimeType(['image/jpeg']))  // JPEG画像のBlob
```

:::message
[Blob](https://developer.mozilla.org/ja/docs/Web/API/Blob) を継承したオブジェクトも Blob として扱われる点に注意してください。
例えば [File](https://developer.mozilla.org/ja/docs/Web/API/File) は Blob を継承しているため、`v.blob()` で検証することができます。
:::


## symbol

```ts
import * as v from "valibot";

v.symbol() // Symbolか判定するスキーマ
```

:::message
特定の Symbol か判定したい場合は `v.literal()` を使用してください。
:::


## instance ( instanceof )

```ts
import * as v from "valibot";

class AnyClass {}

v.instance(AnyClass) // AnyClassにマッチするスキーマ
```

内部的には `instanceof` で判定しているので、`insntanceof` の挙動に注意してください。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/instanceof

## custom

独自の検証をするスキーマを定義するときに使用します。例えば `"hoge"` が文字列の最後にあるか検証するスキーマを作成するには以下のようにします👇

```ts
import * as v from 'valibot';

const schema = v.custom<`${string}にょ`>((input) => {
  return typeof input === "string" && input.endsWith("にょ")
})

type SchemaType = v.InferOutput<typeof schema> 
// `${string}にょ` として型が生成されます
```

:::message
`v.custom()` に型引数に型を渡さなかった場合、`v.InferOutput<T>` で出力される方は `unknown` になってしまう点に注意してください。
:::


## lazy ( 再帰的な型 )

再帰的なスキーマを定義したい時には v.lazy() を使います。

```ts
import * as v from 'valibot';

type BinaryTree = {
  element: string;
  left: BinaryTree | null;
  right: BinaryTree | null;
};

const BinaryTreeSchema: v.GenericSchema<BinaryTree> = v.object({
  element: v.string(),
  left: v.nullable(v.lazy(() => BinaryTreeSchema)),
  right: v.nullable(v.lazy(() => BinaryTreeSchema)),
});
```

:::message
TypeScript の制約により再帰スキーマの `v.InferOutput<T>` などによる型生成はできません。型を付ける場合は `v.GenericSchema<T>` を使って明示的に型を付ける必要があります。
:::

また入力値からスキーマを生成する事もできます。

```ts
import * as v from 'valibot';

// こちらは再帰スキーマでは無いので型生成が可能です
const LazySchema = v.lazy((input) => {
  switch (input) {
    case "foo":
      return v.object({ type: v.literal("foo") });
    case "bar":
      return v.object({ type: v.literal("bar") });
    case "hoge":
      return v.object({ type: v.literal("hoge") });
    default:
      return v.never();
  }
})

type SchemaType = v.InferOutput<typeof LazySchema> 
// { type: "foo"; } | { type: "bar"; } | { type: "hoge"; }
```


## 逆引きチートシートについて

より具体的な実装方法については、ここで紹介すると長くなりすぎるので以前作ったスクラップの方を参照して頂ければと思います🙏

https://zenn.dev/uttk/scraps/3a51a34ddbdc4a


## あとがき

ここまで読んでくれてありがとうございます。

正直に言うと、タイトルと記事の冒頭部分をやりたかっただけで書きました。
後悔はありません。むしろ、爽やかな気持ちでいっぱいです😌

記事の内容については前に書いたスクラップとほとんど同じなんで別に記事を書く必要は無かったんですが、スクラップだと目次が出なくて個人的にちょっと不便だったので記事として書き直した次第です🖊 

逆引きチートシートについては、作ったもののあまり見てないのでそのままにしておきます。
何かあればスクラップの方にコメントしてください。

これが誰かの参考になれば幸いです。
記事に間違いなどがあれば、コメントなどで教えて頂けると嬉しいです。

それではまた 👋