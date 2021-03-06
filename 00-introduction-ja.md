# BOLT #0: イントロダクションと索引

ようこそ、みなさん！これらのライトニングテクノロジー基盤(Basis of Lightning Technology, BOLT)に関するドキュメントは、相互協力によるオフチェーンbitcoin送金のためのレイヤー２プロトコルです。基本的にオフチェーンで行われ、オンチェーントランザクションは必要に応じて送金を執行するときに使われます。

いくつかの要件は分かりにくく捉えにくいものではありますが、ドキュメントにあるもののモチベーションや理由がどんなものなのかが分かりやすくなるように試行錯誤をしてきました。まだまだ分かりにくいところがあると思いますので、もし混乱した点、間違っている点がありましたら我々にご連絡いただき、よりよいものに改善していけるようご協力をお願いします。

このドキュメントのバージョンは０です。

1. [BOLT #1](01-messaging.md): 基本プロトコル
2. [BOLT #2](02-peer-protocol.md): チャネル管理のためのピア間プロトコル
3. [BOLT #3](03-transactions.md): BitcoinトランザクションとScriptフォーマット
4. [BOLT #4](04-onion-routing.md): オニオンルーティングプロトコル
5. [BOLT #5](05-onchain.md): オンチェーントランザクションを操作するための推奨案

7. [BOLT #7](07-routing-gossip.md): P2Pノードとチャネル探索
8. [BOLT #8](08-transport.md): 認証および暗号済み転送
9. [BOLT #9](09-features.md): 割り当て済み特徴フラグ
10. [BOLT #10](10-dns-bootstrap.md): NDSブートストラップと補助ノードの場所
11. [BOLT #11](11-payment-encoding.md): ライトニングペイメントのための請求書

## 閃き: ライトニングの簡単な説明

ライトニングは、チャンネルネットワークを使ってビットコインの高速支払いを実行するためのプロトコルです。

### チャンネル

ライトニングは、チャンネルを開設することで稼働します: 二人の参加者がビットコインネットワークで取得したビットコイン（例えば0.1ビットコイン）を含むライトニングペイメントチャンネルを開きます。それは彼らの署名によってのみ支払われます。

まずはじめに、一つの宛先に対してすべてのビットコインを送信するトランザクションを双方が保持します。双方は、後ほどこの資金を分割するトランザクションに対する署名を実施することが出来ます。
例えば、0.09ビットコインを片方に、0.01ビットコインをもう片方に支払うトランザクションです。そして古いビットコイントランザクションを否認することで古い支払いを止めます。

チャンネルの開設についての詳細は[BOLT #2: Channel Establishment](02-peer-protocol.md#channel-establishment) を、チャンネル作成のためのトランザクションフォーマットの詳細については[BOLT #3: Funding Transaction Output](03-transactions.md#funding-transaction-output)を参照ください。参加者間の否認や失敗で、交差署名付きトランザクションの利用が必要となる点については [BOLT #5: Recommendations for On-chain Transaction Handling](05-onchain.md)をご参照ください。

### 条件付き支払い

ライトニングチャンネルは二人の参加者間での支払いのみを許可しますが、ネットワークのすべての参加者への支払いを許可するためにチャンネルを接続してネットワークを作成することが出来ます。これには、チャンネルに追加することができる条件付き支払い技術（例えば、「６時間以内に秘密を公開したら0.01ビットコインを取得できる」）が必要となります。
参加者が秘密を公開した場合、そのビットコイントランザクションは条件付き支払いがないトランザクションに置き換えられ、その参加者のアウトプットに資金が追加されます。

条件付き支払いの操作については[BOLT #2: Adding an HTLC](02-peer-protocol.md#adding-an-htlc-update_add_htlc)を、ビットコイントランザクションの完全なフォーマットについては[BOLT #3: Commitment Transaction](03-transactions.md#commitment-transaction)をご参照ください。

### 転送

そのような条件付き支払いは安全に相手の参加者に対して短い時間で転送することが可能です。例えば「５時間以内に秘密を公開したら0.01ビットコインを取得できる」という条件付き支払いは、中間者の信頼無しでの複数のチャンネルのチェインによるネットワーク化を可能とします。

転送支払いの詳細については[BOLT #2: Forwarding HTLCs](02-peer-protocol.md#forwarding-htlcs)を、支払指示がどのようにトランスポートされるかは[BOLT #4: Packet Structure](04-onion-routing.md#packet-structure)をご確認ください。

### ネットワーク・トポロジー

支払いを実行するため、参会者はどのチャンネルなら送り届けることができるのかを知る必要があります。
参加者はお互いにチャンネルとノードの作成状況、そしてアップデートを共有します。

相互連絡のプロトコルの詳細については[BOLT #7: P2P Node and Channel Discovery](07-routing-gossip.md)を、ネットワークのブートストラップについては[BOLT #10: DNS
Bootstrap and Assisted Node Location](10-dns-bootstrap.md)を参照ください。

### 支払い請求書
参加者は彼女がいくら払うべきかを知らせるための請求書を受け取ります。

支払者が支払いが正常に完了したことを証明するための送信先と支払い目的を説明するプロトコルについての詳細は[BOLT #11: Invoice Protocol for Lightning Payments](11-payment-encoding.md)をご確認ください。

## 用語集と用語説明

* *Funding Transaction(ファンディングトランザクション)*:
   * チャネル上で両方のピアに対する支払いをするための不可逆なオンチェーントランザクション(2-of-2マルチシグアドレスへの送金)。
     相互同意によってのみ資金を移すことができます。

* *チャネル*:
   * ２つの *ピア* 間で構成される素早くオフチェーンで通貨を相互交換するための方法。
     資金を移動させるために、更新される *コミットメントトランザクション* に対する署名を２つのピアで交換します。

* *Commitment Transaction(コミットメントトランザクション)*:
   * ファンディングトランザクションを使うためのトランザクション。
     それぞれのピアは、相手のピアからもらったこのコミットメントトランザクション用の署名を保持しておきます。
     このため、この署名にはそれと紐づいて使用できるコミットメントトランザクションが常にあります。
     新しいコミットメントトランザクションが２つのピア間で作られると、古いコミットメントトランザクションは *無効* になります。

* *HTLC*: Hashed Time Locked Contract(資金のロック期限が設けられたハッシュを伴う契約)
   * ２つのピア間の条件付き支払い。
     受金者は署名と *ペイメントプレイメージ* を提示することでこの支払いを使うことができ、また支払者はあらかじめ決められた期限以降であればこの契約をキャンセルすることができます。
     これらは *コミットメントトランザクション* のアウトプットとして実装されます。

* *Payment hash, payment preimage(ペイメントハッシュ、ペイメントプレイメージ)*:
   * HTLCにはペイメントハッシュが含まれています。
     このペイメントハッシュはペイメントプレイメージのハッシュ値です。
     最終的な受金者だけがペイメントプレイメージを知っています。
     受金者が資金を受け取るためにはこのプレイメージを公開する必要があるため、このプレイメージが公開されているということは受金者が支払いを受け取ったことの証明と考えられます。

* *Commitment revocation key(コミットメント取り消し鍵)*:
   * 全ての *コミットメントトランザクション* には一意の *コミットメント取り消し鍵* が紐付いています。
     相手のピアはこの鍵を使うことで全てのアウトプットを直ちに使うことができます。
     つまり、この鍵の公開は全てのコミットメントトランザクションの取り消しを意味します。
     これを行うために、それぞれのアウトプットはコミットメント取り消し鍵に対応するコミットメント取り消し公開鍵を参照しています。

* *Per-commitment secret(個別コミットメントシークレット)*:
   * 全てのコミットメントは *個別コミットメントシークレット* から導出された鍵と紐付きます。
     このシークレットを使うことで、以前の全コミットメントに対する複数の個別コミットメントシークレットをコンパクトに保存できるようになっています。

* *Mutual Close(相互的クローズ)*:
   * チャネルの協力的クローズ。
     これは *ファンディングトランザクション* の無条件で使用するトランザクションのブロードキャストによって行われます。
　　 このトランザクションにはそれぞれのピアへのアウトプット(アウトプットの中にとても小さい金額のものがなければ。もしあれば片方だけ。)を持ちます。

* *Unilateral Close(一方的なクローズ)*:
   * チャネルの非協力的なクローズ。
     これは *コミットメントトランザクション* をブロードキャストすることで行われます。
     このトランザクションは相互的クローズのトランザクションと比べるとサイズが大きいトランザクション(つまり、非効率的)で、コミットメントをブロードキャストしたピアは事前に決められた期日まで自身のアウトプットにアクセスできません。

* *Revoked Transaction Close(取り消し済みトランザクションのクローズ)*:
   * チャネルの不正なクローズ。
     これは、取り消された *コミットメントトランザクション* がブロードキャストされることによって行われます。
     ブロードキャストされた相手のピアが *コミットメント取り消し秘密鍵* を知っていれば、このピアは *ペナルティートランザクション* を作ることができます。

* *Penalty Transaction(ペナルティートランザクション)*:
   * 取り消し済みコミットメントトランザクションの全てのアウトプットを使うトランザクション。
     このトランザクションの作成時に *コミットメント取り消し秘密鍵* を使います。
     もし相手のピアが取り消し済み *コミットメントトランザクション* をブロードキャストして "ズル" をしようとする場合にこれを使います。

* *Commitment Number(コミットメント番号)*:
   * *コミットメントトランザクション* それぞれに対して順番に振られる４８ビットの番号。
     このカウンターは０から始まり、チャネルの中でそれぞれのピアごとに独立に持つ番号です。

* *It's ok to be odd*:
   * いくつかの数値フィールドに対して適用されるルール。
     これらのフィールドは任意または強制のいずれかを示します。
     偶数番号のfeatureは両端ノードが問題になっているfeatureをサポートしておかなければならないことを示し、一方奇数番号のfeatureは片端のノードによってfeatureが無視されるかもしれないことを示します。

## Theme Song


      Why this network could be democratic...
      Numismatic...
      Cryptographic!
      Why it could be released Lightning!
      (Release Lightning!)


      We'll have some timelocked contracts with hashed pubkeys, oh yeah.
      (Keep talking, whoa keep talkin')
      We'll segregate the witness for trustless starts, oh yeah.
      (I'll get the money, I've got to get the money)
      With dynamic onion routes, they'll be shakin' in their boots;
      You know that's just the truth, we'll be scaling through the roof.
      Release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      [Chorus:]
      Oh released Lightning, it's better than a debit card..
      (Release Lightning, go release Lightning!)
      With released Lightning, micropayments just ain't hard...
      (Release Lightning, go release Lightning!)
      Then kaboom: we'll hit the moon -- release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      We'll have QR codes, and smartphone apps, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      P2P messaging, and passive incomes, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      Outsourced closure watch, gives me feelings in my crotch.
      You'll know it's not a brag when the repo gets a tag:
      Released Lightning.


      [Chorus]
      [Instrumental, ~1m10s]
      [Chorus]
      (Lightning! Lightning! Lightning! Lightning!
       Lightning! Lightning! Lightning! Lightning!)


      C'mon guys, let's get to work!


   -- Anthony Towns <aj@erisian.com.au>


## Authors


[ FIXME: Insert Author List ]


![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
