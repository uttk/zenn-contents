---
title: "無駄に話が長い人を無駄に邪魔する無駄なChrome拡張"
emoji: "🧇"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["webrtc", "chromeextension"]
published: true
---

# この記事について

https://twitter.com/uttk_dev/status/1468491788465803264

この記事では上記の Chrome 拡張で使用している技術や知見について解説していきたいと思います。
記事中のソースコードは TypeScript を用いて開発していきますので、予めご了承ください。

また拡張のソースコードは以下のリポジトリに公開していますので、参考にして下さい 👇

https://github.com/uttk/ojaman

さて、それでは解説に行きましょう 🚀

# MediaStream と Track について

:::message
この解説には簡略化のために厳密には正しくない表現が含まれています。
もし正しい情報を知りたい場合は、[#参考にした記事](#参考にした記事) のセクションに載せてある記事をご参照ください 🙇‍♂️🙇‍♀️
:::

動画を差し込む処理を見て行く前に、MediaStream と Track の関係について簡単に解説していきたいと思います。以下の図を見てください 👇

![WebRTCの簡易的なデータフロー図](/images/ojaman-chrome-extensions/web-rtc-flow.png)

上記の図は、WebRTC(ビデオ通話の通信フロー)を簡単に表したモノです。
上記の図から分かる通り、MediaStream を RTCPeerConnection のインスタンスに渡し、双方で Track を通信し合ってビデオ通話を実現しています。

この時の Track がカメラ情報やマイク情報を格納したデータとなっており、今回の Chrome 拡張でも、基本的には Track や MediaStream を使って動画を通話中に差し込みますので、上記のような構造を理解しておくと、この後の処理が理解しやすくなると思います 🚏

# replaceTrack() で動画を差し込む技術

ビデオ通話中に動画を差し込むには、送信する Track を置き換える(replace)する必要があり、それを行う API が `RTCRtpSender.replaceTrack()` です。

https://developer.mozilla.org/en-US/docs/Web/API/RTCRtpSender/replaceTrack

この API を使用することで、通話中に送信するカメラ・マイク情報の代わりに動画の情報を相手に送りつけることができます。使用方法は以下の通りです 👇

```ts:送信するTrackを別のTrackに置き換える処理
const video = document.getElementById("insert-video"); // 挿入する動画のvideoタグ
const stream: MediaStream = video.captureStream(); // 動画情報を取得する
const tracks: MediaStreamTrack[] = stream.getTracks(); // 映像と音声情報を取得する

const pc = new RTCPeerConnection(); // 通信情報を扱うためのインスタンス
const senders = pc.getSenders(); // 送信情報を扱うためのインスタンス

// 以下の処理で、相手側に動画情報を送るようにする
for(const track of tracks) {
  for(const sender of senders) {
    // 送信するTrackの種類("video"・"audio")を判定する
    if(sender.track?.kind === track.kind) {
      await sender.replaceTrack(track); // 同じ種類ならTrackを置き換える
    }
  }
}
```

:::message
`captureStream()` は、Firefox では `mozCaptureStream()` となるので注意が必要です！
参照：[HTMLMediaElement.captureStream() - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/HTMLMediaElement/captureStream)
:::

**注意点として、自分側のプレビュー画面には動画は表示されません！**
そのため、自分側の画面にプレビューを表示するためには別途処理が必要ですが、やる事に対してコードが凡雑であるため、記事の最後辺りに後述したいと思います 🙏

しかし、基本的に上記の処理を行うことで、通話中に動画を差し込むことが可能ですし、上記と逆の処理をすることで元に戻すことが可能です。結構、簡単ですね ✨
しかし、実際の上記の処理をやろうとすると一つ問題点があります。

**それは `RTCPeerConnection` のインスタンスが、そもそも取得できないという事です。**

この Chrome 拡張は、既に通信している所に動画を差し込むため、`RTCPeerConnection` のインスタンスは Chrome 拡張側で作成できません。そのため、 `RTCPeerConnection` のインスタンスを取得する為の処理を書く必要があります。

次の節で、その処理を見て行きましょう 🗺

# RTCPeerConnection のインスタンスを横取りする技術

`RTCPeerConnection` のインスタンスを取得するためには、コンストラクタを上書きして、そこからインスタンスを取得する処理を実装します。ソースコードを見てみましょう 👇

```ts:RTCPeerConnectionクラスのconstructorを上書きする
/** RTCPeerConnectionインスタンスの配列 */
const connections: RTCPeerConnection[] = [];

/**
 * @description コンストラクタを上書きするためのクラス
 */
export class MyRTCPeerConnection extends RTCPeerConnection {
  constructor(...args: any[]) {
    super(...args);

    // RTCPeerConnectionインスタンスを保存する
    connections.push(this);
  }

  // RTCPeerConnectionインスタンスを返す関数
  static getConnectedConnection(): RTCPeerConnection | null {
    return connections.find((v) => v.connectionState === "connected") || null;
  }
}

// 自作したクラスを RTCPeerConnection とする
window.RTCPeerConnection = MyRTCPeerConnection;
```

ブラウザ拡張では、**表示されるサイトよりも早くスクリプトを実行する事ができる**ため、
あるサイト(GoogleMeet など)が `new RTCPeerConnection()` を実行する前に上記のスクリプトを実行する事ができます。

これによって、上書きされたサイトでは `MyRTCPeerConnection` のコンストラクタを経由して、`RTCPeerConnection` のインスタンスを取得することになるため、`RTCPeerConnection` のインスタンスを必ず `connections` に追加することができます。

これにより、いつでも `RTCPeerConnection` のインスタンスを取得することができます 👇

```ts:通信中のRTCPeerConnectionのインスタンスを取得する
const pc: RTCPeerConnection | null = MyRTCPeerConnection.getConnectedConnection();

/* -- RTCPeerConnectionを使った処理 -- */
```

# 会話時間を計測する技術

今回の Chrome 拡張では、自分の話している時間を計測していますが、この時間を計測する処理は [hark](https://github.com/otalk/hark) と言うライブラリを使用して計測しています。

https://github.com/otalk/hark

この [hark](https://github.com/otalk/hark) は、音声情報から自分が話しているかを検出してくれるライブラリで、このライブラリで `'speaking'` イベントが取得できますので、その時に時間を計測するようにしています。具体的なソースコードは以下のようになります 👇

```ts:会話時間を計測する処理
// マイク情報を取得する
const mic: MediaStream = navigator.mediaDevices.getUserMedia({ audio: true });

// 会話監視をするためのインスタンスを取得
const speechEvents = hark(mic);

// しゃべり始めに一回だけ実行されるイベント
speechEvents.on("speaking", () => {
  /* -- ここで時間計測開始の処理 -- */
});

// しゃべり終わりに一回だけ実行されるイベント
speechEvents.on("stopped_speaking", () => {
  /* -- ここで時間計測終了の処理 -- */
});
```

注意点としては、`"speaking"` イベントはしゃべり始めの一回しか実行されませんので、`"stopped_speaking"` イベントを使ってタイマーを止める処理をする必要があります。

## hark の仕組み

さて、結構簡単に使える [hark](https://github.com/otalk/hark) ですが、実はソースコードはとても少なく、単純なモノとなっています。そのため、音声の検出方法も単純で、[Web Audio API](https://developer.mozilla.org/ja/docs/Web/API/Web_Audio_API) を使用して、音量が設定された閾値(dB)よりも高ければ、しゃべっているとみなします。なので、物音などでも `"speaking"` イベントが実行されてしまうことに注意してください！

より詳細な挙動は、以下のサイトを参照して頂けると幸いです 👇

https://github.com/otalk/hark/blob/master/hark.js

https://developer.mozilla.org/ja/docs/Web/API/Web_Audio_API

https://developer.mozilla.org/en-US/docs/Web/API/AnalyserNode/getFloatFrequencyData

# その他の知見について

その他、細々とした知見を以下に共有したいと思います 📝

## captureStream()を使う時の注意

https://developer.mozilla.org/ja/docs/Web/API/HTMLMediaElement/captureStream

`<video />` から MediaStream を取得する為に使用している `captureStream()` ですが、現時点(2021/12/07)で、恐らくブラウザ側のバグがあり、実行すると動画の音声が機能しなくなる現象が発生しています。このバグは Chrome ・Firefox 双方で発生しているため、使用する際には注意が必要です。※ 一部のバージョンでは発生しないようです。

詳細については以下の issue を参照してください 👇

https://github.com/webrtc/samples/issues/1412

## captureStream()がデフォルトの型に無い時

TypeScript を使用して開発している場合、`captureStream()` が `HTMLMediaElement` の型に定義されてない場合があります。その場合は、ソースコード中か `*.d.ts` ファイルに、以下の型定義を記述してあげることで型エラーを回避できます 👇

```ts:HTMLMediaElementの型定義を拡張する
// captureStream API が型定義に無いので上書きして追加する
declare global {
  interface HTMLMediaElement {
    captureStream?(): MediaStream;
    mozCaptureStream?(): MediaStream;
  }
}
```

## `HTMLVideoElement.seeking` は信用できない

今回の Chrome 拡張では、動画が一時停止された時に動画の差し込みを終了する処理を実行しているのですが、何も対策をしていないと、その処理がシークバーを動かしている時にも発生します。これを避けるために、`HTMLVideoElement.seeking` を使ってシークバーを動かしている時は処理しないようにしているのですが、どうも `HTMLVideoElement.seeking` の値が正しく取得できないことが稀に良くありました。

これの回避策として、二段階チェックを行うようにして、なるべく誤判定を避けるようにしています 👇

```ts:渡されたvideo要素がシーキングしているか判定する
const isSkeeking = (video: HTMLVideoElement): Promise<boolean> => {
  if (video.seeking) return Promise.resolve(true);
  return new Promise((resolve) => setTimeout(resolve, 50, video.seeking));
};
```

ただ、これを使用しても度々誤判定してしまっているので、もっと何か対策が必要だと思います。

## 送信元のプレビュー画面に動画を挿入する

[#replaceTrack() で動画を差し込む技術](<#replacetrack()-で動画を差し込む技術>)のセクションでは、相手側に表示される動画情報を変更することはできましたが、自分自身(送信元)のプレビュー画面に表示される動画情報は変更されません。そのため、送信元のプレビュー画面にも同じように動画を差し込むには別途処理が必要です。

しかし、この処理は意外と大変です。理由としては、

1. video タグが複数あった場合、プレビュー画面の video タグを特定する必要がある
2. [MediaStream.clone()](https://developer.mozilla.org/ja/docs/Web/API/MediaStream/clone) などにより複製された場合、MediaStream を取得するのが大変になる

の二点が挙げられます。

この問題を解決するために、「 ページ内の全ての video タグから MediaStream を取得し、その MediaStream の中から強引に変更したい Track を取得して変更する 」という力業で解決しています。

言葉だけだと分かりにくいと思いますので、具体的なソースコードを以下に示します 👇

```ts:送信元のプレビュー画面に動画を差し込む処理
const insertVideo = document.getElementById("insert-video"); // 挿入する動画のvideoタグ
const stream: MediaStream = video.captureStream(); // 動画情報を取得する
const insertTracks: MediaStreamTrack[] = stream.getTracks(); // 映像と音声情報を取得する

// 置き換えたい Track 情報を取得する
const cameraTracks: MediaStream = navigator.mediaDevices
  .getUserMedia({ video: true })
  .getTracks();

// ページ内の全てのvideoタグを取得
const videos = Array.from(document.getElementsByTagName("video"));

// video タグからカメラ情報を表示している MediaStream を取得
videos.forEach((video) => {
  const stream = video.srcObject;

  if (!(stream instanceof MediaStream)) return;

  // カメラ情報の Track を持っているか判定する
  const tracks = stream.getTracks();

  for (const track of tracks) {
    for (const cameraTrack of cameraTracks) {
      // 動画を差し込むべきか判定する
      if (
        track.kind === cameraTrack.kind &&
        track.label === cameraTrack.label
      ) {
        // 現在表示されているTrackをvideoタグのMediaStreamから削除
        tracks.forEach((track) => stream.removeTrack(track));

        // 動画のTrackをvideoタグのMediaStreamに追加する
        insertTracks.forEeach((track) => stream.addTrack(track));

        // 動画を差し込んだらループを終了する
        return;
      }
    }
  }
});
```

はい、見て分かる通り、かなり面倒くさいです 😅
元に戻す時も、逆の処理を行う必要があるので、面倒臭ささがさらに増します()

**ただ注目すべき点があります。** それは、送信する時と違って MediaStream の Track を変更するだけで済む点です。そのため、差し込みたい MediaStream と Track があれば比較的簡単に実装できますし、**既に再生されている video タグを止めずに動画を変更することが可能です。**

使う場面は少ないと思いますが、覚えておくと何処かで役立つかもしれませんね！
( こうして無駄知識が増えていくのであった... )

# 参考にした記事

今回の Chrome 拡張を作るにあたり、以下のサイト・記事を参考にさせて頂きました 🙇‍♂️🙇‍♀️

https://webrtcforthecurious.com/ja/

https://www.hiramine.com/programming/videochat_webrtc/index.html

https://webrtc.org/getting-started/overview

https://zenn.dev/voluntas/scraps/82b9e111f43ab3

https://developer.mozilla.org/ja/docs/Web/API/WebRTC_API

# あとがき

ここまで読んでくれてありがとうございます 🙏

私事ですが、最近ビデオ通話をする機会が多くなり、その中でついつい自分の話を長々としてしまうことが多々あり、ビデオ通話が終わった後に後悔してしまうことがありました。なので、今回の Chrome 拡張のようなモノを作れば、誰も傷つけずに楽しく長話をするのを辞めさせられるのではないかと思い、今回作ってみました。

しかし、実際使ってみると音ズレが酷かったりして、ちゃんと使えるようにするには、まだまだ改良しないといけないのですが、やりたいことは達成できたので、一区切りという事で多めに見てやって下さい...

もし分からない点や記事に間違いなどがあれば、コメントや Twitter などで教えて頂けると嬉しいです。
これが誰かの参考になれば幸いです。

それではまた 👋

#### 後日談というか今回のオチ

さて、そろそろこの記事を描き上げてしまおう。
ブラウザが音をたてている。何かビデオ通話の参加リクエストが来た時のような音を。
しかしブラウザのセキュリティを押し破ったところでわたしを見つけられはしない。

いや、そんな！　あの手は何だ！　画面に！　画面に！
