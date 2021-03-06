# BOLT #1: 基本プロトコル

## 概要

このプロトコルは、個々のメッセージが従う基礎的な認証および整列転送メカニズムがあることを仮定しています。
[BOLT #8](08-transport.md) はライトニングネットワークで使われる標準的な転送レイヤーの記述に特化しており、上記の保証を満たすどんな転送方法で下位層を置き換えても問題ないように設計されています。

デフォルトのTCPポートは9735番を使います。これは、16進数での `0x2607` に対応し、LIGHTNINGに対するユニコードの符号位置になっています。<sup>[1](#reference-1)</sup>

全てのデータフィールドは特に指定がない限りビッグエンディアンです。

## 目次
  * [コネクションハンドリングと多重化](#connection-handling-and-multiplexing)
  * [ライトニングメッセージフォーマット](#lightning-message-format)
  * [セットアップメッセージ](#setup-messages)
    * [ `init` メッセージ](#the-init-message)
    * [ `error` メッセージ](#the-error-message)
  * [コントロールメッセージ](#control-messages)
    * [ `ping` メッセージと `pong` メッセージ](#the-ping-and-pong-messages)
  * [謝辞](#acknowledgements)
  * [参考文献](#references)
  * [著者](#authors)

## コネクションハンドリングと多重化

１つのピアおよびチャネルメッセージ(チャネルidを含む)に対して１つのコネクションを使うように実装しなければいけません。
チャネルメッセージは１つのコネクション上で多重化されて実装されなければいけません。

## ライトニングメッセージフォーマット

全てのライトニングメッセージは復号後に以下のフォーマッになります。

1. `type`: メッセージの種類を示す２バイトフィールド(ビッグエンディアン)
2. `payload`: 可変長ペイロード。これはメッセージからtypeを除いた部分であり、 `type` にマッチしたフォーマットに従います。

`type` フィールドは `payload` フィールドを解釈する方法を示します。
個々のtypeに対するフォーマットはこのリポジトリの仕様の中で規定されています。
typeフィールドは _it's ok to be odd_ ルールに従っています。
このため、ノードは受金者がそれに対応しているかどうかを把握することなく奇数番号のtypeを送信するかもしれません。
ノードは事前の確認なしにここに挙げられていない偶数番号のtypeを送信してはいけません。
もしtypeが奇数番号であれば、ノードは知らないタイプのメッセージを無視しなければいけません。
もしtypeが偶数番号であれば、ノードは知らないタイプのメッセージを受け取ると同時にチャネルを停止しなければいけません。

メッセージは、最重要ビットによって論理的には以下の４つのグループに分けられます。

 - セットアップ & コントロール (type `0`-`31`): コネクションセットアップ、コントロール、サポートしている機能、エラーレポーティングに関連したメッセージ。詳細は以下に記載。
 - チャネル (types `32`-`127`): マイクロペイメントチャネルのオープンとクローズに使われるメッセージが含まれています。詳細は [BOLT #2](02-peer-protocol.md) に記載。
 - コミットメント (types `128`-`255`: 現在のコミットメントトランザクションの更新に関連したメッセージが含まれています。コミットメントトランザクションの更新には、手数料の更新、署名の交換だけでなく、追加、取り消し、HTLCの確定(ブロードキャスト)も含まれます。詳細は [BOLT #2](02-peer-protocol.md) に記載。
 - ルーティング (types `256`-`511`): アクティブなルート探索だけでなく、ノードやチャネル関係の通知も含まれます。詳細は [BOLT #7](07-routing-gossip.md) に記載。

メッセージのサイズはトランスポート層の制限により符号なし２バイト整数に収まる必要があります。
このため、最大サイズは65535バイトになります。
ノードはメッセージ内にあるtypeごとに想定される長さよりも長い食み出たデータは無視しなければいけません。
また、もし規定に対して不十分な長さの既知メッセージを受け取った場合、ノードはチャネルを停止しなければいけません。

### 合理性

標準的な `SHA2` やBitcoinの公開鍵のエンコーディングはビッグエンディアンです。
このため、ライトニングネットワークでの他のフィールドで他のエンコーディングを使うことは統一性に欠けると思われます。

長さは暗号学的変換によって65535バイトに制限され、プロトコル内のメッセージはこの長さを超えることはあり得ません。

"it's OK to be odd" ルールは、将来的にクライアントサイドでの特別な実装を可能とし調整なしに拡張可能にするためにあります。
"ignore additional data" ルールも同様に将来における拡張性のために設けられています。

実装としては、８バイト境界アラインメント(ここにあるtypeで最も大きく自然なアラインメント制約)でのメッセージデータを持つのが好ましいかもしれません。
もしtypeフィールドが無駄だと考えられる場合は６バイトパディングを追加します。
つまり、アラインメントはメッセージを復号し６バイトのパディングとともにバッファに入れることで行われる可能性があります。


## セットアップメッセージ

### `init` メッセージ

認証が完了すると、これが再接続だとしてもこのノードでサポートしているまたは必須となっているfeatureが最初のメッセージで通知されます。
奇数番号featureは任意、偶数番号featureは強制です( _it's OK to be odd_ を意味します) 。
これらのビットの意味は将来的に以下で定義されます。

1. type: 16 (`init`)
2. data:
   * [`2`:`gflen`]
   * [`gflen`:`globalfeatures`]
   * [`2`:`lflen`]
   * [`lflen`:`localfeatures`]

２バイトの `gflen` フィールドと `lflen` フィールドはすぐあとに続くフィールドのバイト長を示しています。

#### 要件

メッセージを送るノードは `init` メッセージを最初のライトニングメッセージとして送らなければいけません。
メッセージを送るノードはfeatureフィールドを表す最小限の長さを使うべきです。
このノードは、ピアがサポートする必要があるfeatureに対応したfeatureビットはセットしなければいけません。また、任意にサポートするfeatureに対応したfeatureビットはセットしたほうが望ましいです。

もし理解できない偶数ビットを伴った `globalfeatures` または `localfeatures` を受け取った場合、メッセージを受け取ったノードはチャネルを停止しなければいけません。

それぞれのノードは他のメッセージを送る前に `init` メッセージを受け取るまで待たなければいけません。

#### 合理性

偶数番号featureまたは奇数番号featureの意味付けは、前方互換性のない変更および後方互換性のある変更を可能にします。
一般的に、任意のfeatureがのちに強制的なものにできるように、ビットはペアになるように揃えられているほうがよいです。

featureに互換性がないと誤判断しないように、ノードは他のfeatureの状況も見ます。

featureマスクは２つのノード間でのプロトコルにのみ影響を与えるような局所的featureとHTLCに影響を与え他のノードにも通知されるような全域的featureに分けます。

### `error` メッセージ

判断の簡易化のため、ピアに何かが間違っていますよと頻繁に教えてあげることは意味があります。

1. type: 17 (`error`)
2. data:
   * [`32`:`channel_id`]
   * [`2`:`len`]
   * [`len`:`data`]

２バイトの `len` フィールドはすぐあとに続くフィールドのバイト長を示します。

#### 要件

`channel-id` フィールドがゼロ(つまり、全てのバイトがゼロ)でない限り、チャネルは `channel-id` フィールドによって指定されます。 `channel-id` フィールドがゼロのときは全てのチャネルのことを意味します。

ノードは、プロトコル違反であったり、チャネルを使えなくするまたは追加のコミュニケーションを不可能にする内部エラーに対して `error` メッセージを送るべきです。
ノードはdataフィールドが空のerrorメッセージを送る可能性もあります。
`error` メッセージを送るノードは、errorメッセージによって指定されるチャネルを停止しなければいけません。
また、もし `channel-id` フィールドがゼロであれば全てのチャネルを停止し、そのコネクションを閉じなければいけません。
ノードは `len` フィールドに `data` フィールドの長さと等しい値を入れなければいけません。
もし不正な署名チェックが原因で失敗した時には、ノードは、`funding_created`メッセージ、`funding_signed`メッセージ、`closing_signed`メッセージ、`commitment_signed`メッセージに対する返答に１６進数のrawトランザクションを含めるべきです。

`error` メッセージを受け取ったノードは、そのメッセージで指定されているチャネルを停止しなければいけません。
もし `channel-id` フィールドがゼロであれば、全てのチャネルをを停止しそのコネクションを閉じなければいけません。
メッセージで指定されているチャネルが存在しない場合、そのメッセージを無視しなければいけません。
もし受け取ったerrorメッセージがより大きいものであれば、 `len`フィールドで指定されている長さにパケットを切り落として使わなければいけません。

もし `data` フィールドの中の文字列が単に表示可能なASCII文字で構成されている場合、受け取ったノードは `data` フィールドの中身を言葉通りに表示するだけするべきです。

#### 合理性

単にコネクションが切れてしまったのであればピアが再度コネクションを確立しなおせばいいのですが、それもできないようなコミュニケーションを諦めざるをえない回復不可能なエラーはありえます。
またこのような状況はピアにバグがあることを示すこともあるため、診断のためにプロトコル違反があったときの対応方法を決めておくことは有用です。

## コントロールメッセージ

### `ping`メッセージと`pong`メッセージ

とても長く張られ続けるTCPコネクションを許可するために、両端ノードが時にはアプリケーションレベルでTCPコネクションをアクティブにし続ける必要があるかもしれません。
そのようなメッセージはトラフィックパターンを読みにくくしてしまう可能性もあります。

1. type: 18 (`ping`)
2. data:
    * [`2`:`num_pong_bytes`]
    * [`2`:`byteslen`]
    * [`byteslen`:`ignored`]

`pong` メッセージは `ping` メッセージを受け取った時にいつでも送られます。
これは返答として送られるものですが、また受信側がまだアクティブであることを他方に明示的に通知することでコネクションがアクティブであると通知する意味もあります。

1. type: 19 (`pong`)
2. data:
    * [`2`:`byteslen`]
    * [`byteslen`:`ignored`]

#### 要件

`pong` メッセージまたは `ping` メッセージを送ったノードは `ignored` フィールドにゼロを入れるべきであり、 `ignored` フィールドに機密情報や初期化されたメモリの位置などセンシティブなデータを入れるべきではありません。

ノードは `ping` メッセージを３０秒毎に１回以上送るべきではありません。
pingメッセージに対応した `pong` メッセージがない場合はコネクションを閉じる可能性があります。
この場合チャネルを停止してはいけません。

もし３０秒毎に１つよりも著しく多くの `ping` メッセージを受け取った場合、 `ping` メッセージを受け取ったノードはチャネルを停止するべきです。
もしそうでない場合、 `num_pong_bytes` フィールドが65532より小さければそのノードは `num_pong_bytes` フィールドと等しい `byteslen` を持つ `pong` メッセージを送ることで返答しなければいけません。また、 `num_pong_bytes` フィールドが65532以上であれば `ping` メッセージを無視しなければいけません。

もし `byteslen` フィールドが過去に送ったどの `ping` メッセージの `num_pong_bytes` フィールドにも一致しなければ `pong` メッセージを受け取ったノードはチャネルを停止する可能性があります。

### 合理性

最も大きいメッセージは65535バイトであるため、 `pong` メッセージのtypeフィールドと `bytelen` フィールドそのものを計算に入れると最大の `byteslen` フィールドの値は65531です。
これは、pingメッセージへの返答としてpongメッセージを送るべきではない場合の `nom_pong_bytes` フィールドに対する基準になります。

ペイメントチャネルはいつまで使われるかわからないため、ノード間のコネクションはとても長く維持される可能性があります。
しかし、きっとコネクションを張っているかなりの期間で新しいデータは交換されないです。
しかも、いくつかのプラットフォームではライトニングクライアントは事前予告なく睡眠状態に陥る可能性もあります。
結果として、相手サイドとのコネクションがアクティブであるかを調べるためやすでに確立されたコネクションをアクティブに保つために、一個一個が離れたpingメッセージを使うことになります。

加えて、送信者が受信者に特定のバイト長のレスポンスを要求することは、ネットワーク上のノードに _偽の_ トラフィックを作らせることになります。
それぞれのチャネルに対してなんの更新もすることなくノードの典型的なトラフィックパターンを偽ることができるので、部分的にはパケットおよびタイミング分析からノードを守るために使うことができます。

[BOLT #4](https://github.com/lightningnetwork/lightning-rfc/blob/master/04-onion-routing.md) で定義されているオニオンルーティングプロトコルと結びつけることで、注意深く統計的に作られた偽りのトラフィックはネットワーク内にいる参加者のプライバシーをさらに強化することになります。

いくつかの注意は `ping` メッセージの濫用に対する対策として推奨されるものですが、ネットワーク遅延を防止するためいくらかの自由度が与えられています。
入ってくるトラフィックの濫用に対しては他にも方法はあることを強調しておきます(例えば、奇数番号の未知メッセージtypeを送ることや全てのメッセージに対して最大限パディングをするなど)。

最後に、周期的な `ping` メッセージの使用は [BOLT #8](https://github.com/lightningnetwork/lightning-rfc/blob/master/08-transport.md) で記載しているように頻繁な鍵の使い回しに繋がります。


## 謝辞

TODO(roasbeef); fin

## 参考文献
1. <a id="reference-2">http://www.unicode.org/charts/PDF/U2600.pdf</a>

## Authors

## 著者

FIXME

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
