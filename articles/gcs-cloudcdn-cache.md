---
title: "Cloud Storage + ロードバランサ + Cloud CDN で静的コンテンツを配信する際のキャッシュ設定をおさらい"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

本記事は [Google Cloud Japan Advent Calendar 2022](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370) の [通常版](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370#%E9%80%9A%E5%B8%B8%E7%89%88) の 19 日目の記事です。

Google Cloud Storage (GCS) では、Web フロントエンドなどの静的コンテンツを、バケットに保存するだけで簡単に公開することができて便利です。
さらに、Google Cloud の HTTP(S) Load Balancing と組み合わせることで、HTTPS に対応させたり、Cloud CDN と組み合わせることで CDN にキャッシュすることもできるので、アクセス数の多い商用サービスなどでも充分に対応できます。

一方で、このやり方を採用してシステム設計する際に、ややこしいのがキャッシュの設定です。
特に上記のように LB や CDN と組み合わせた際には、以下のようにキャッシュに関連する設定が複数あるため、挙動が少しわかりにくいです。

- GCS の組み込みキャッシュ設定
- Cloud CDN の TTL 設定
- Cloud CDN の Serve while stale (Stale-while-revalidate) 設定

本記事では、それぞれの設定がどのような挙動になるのか説明します。


## 先に結論

Cloud CDN のキャッシュモードを `FORCE_CACHE_ALL` とした場合を想定すると、最初にクライアントと CDN でキャッシュが作られたあとの各タイミングでクライアントが参照する先は以下のようになります。

![](/images/gcs-cloudcdn-cache/cache-diagram.png)

こう見ると、いろいろな設定が絡んできてなかなかややこしく見えてしまいますね。
ここからは、上記の図を読み解いていくために、各設定項目がどういうものかを説明していきます。

（もしこれだけ見て、「まあそうだよね」と理解できた方は、ここで読み終えていただいても大丈夫です！）


## GCS の組み込みキャッシュ設定

まず前提として、GCS 自体が独自の組み込みキャッシュ機能を備えており、簡易的な CDN のように利用できるようになっています。

> 一般公開オブジェクトはデフォルトで Cloud Storage ネットワークのキャッシュに保存されるため、特に設定をしなくても、Cloud Storage はコンテンツ配信ネットワーク（CDN）のように動作します。

https://cloud.google.com/storage/docs/caching?hl=ja


このキャッシュの設定は、GCS バケットの中の各オブジェクトにメタデータを設定することで `Cache-Control` ヘッダを通じてコントロールできます。

[公式ドキュメント](https://cloud.google.com/storage/docs/metadata?hl=ja#cache-control)に記載されているように、`Cache-Control` メタデータには、以下のような値を指定できます。

- public: オブジェクトを任意のキャッシュに保存できます。
- private: オブジェクトは、リクエスト元のローカル キャッシュに保存されます。
- no-cache: オブジェクトは、キャッシュに保存されますが、Cloud Storage によって最初に検証されない限り、将来のリクエストには使用されません。
- no-store: オブジェクトはキャッシュに保存されません。
- max-age=TIME_IN_SECONDS: キャッシュに保存されたオブジェクトが古くなったとみなされるまでの時間。max-age には任意の長さの時間を設定できます。特別な状況を除いて、古くなったオブジェクトは、キャッシュから提供されません。


重要な点としては、公開バケットで **明示的に指定しなかった場合には `Cache-Control: public, max-age=3600` がデフォルト値として勝手に設定される** ようになっているため、1 時間（＝3,600 秒）の間は GCS のオブジェクトが上書き更新された場合でも、基本的に組み込みキャッシュから古いオブジェクトが返ることになります。
オブジェクトを更新した際にエンドユーザのクライアントに短時間で反映させたい場合には、`max-age` を短く設定したり、`no-store` に設定して GCS の組み込みキャッシュを無効化しましょう。


## Cloud CDN のキャッシュ設定

GCS 単独で HTTP の静的サイトを公開できるとはいえ、実際には、GCS で静的コンテンツを配信する際には、HTTP ではなく HTTPS で配信するために、HTTP(S) Load Balancing と組み合わせることが多いと思います。
その場合、HTTP(S) Load Balancing の設定で Cloud CDN との連携機能を有効にするだけで、CDN 経由で低レイテンシでのコンテンツ配信も行うことができます。

…と聞いて「おおそれは便利だ！」ということで、Cloud CDN - HTTP(S) LB - GCS という構成にする場合、以下のような設定がキャッシュの挙動に関わってきます。

- キャッシュモード（`CACHE_ALL_STATIC` / `USE_ORIGIN_HEADERS` / `FORCE_CACHE_ALL` のいずれか）
- 各種 TTL 設定（`Default TTL` / `Max TTL` / `Client TTL` のそれぞれ）
- Serve stale content (`Stale-while-revalidate`) の設定

なんだかいろいろありますね。
それぞれどんなものか説明します。


### キャッシュモードとは

Cloud CDN を有効にする際には、以下いずれかのキャッシュモードを指定します。

|キャッシュモード|説明|
|----|----|
|`CACHE_ALL_STATIC`|デフォルトの設定。[静的コンテンツとして定義されている Content-Type のファイル（画像や JavaScript、CSS ファイルなど）](https://cloud.google.com/cdn/docs/caching?hl=ja#static)はキャッシュされますが、それ以外のファイル（HTML や JSON など）はキャッシュされません。キャッシュ対象の `Content-Type` でも、`Cache-Control` が `private` や `no-store` に設定されている場合や、`Set-Cookie` ヘッダがついている場合にはキャッシュの対象外となります。|
|`USE_ORIGIN_HEADERS`|オリジン側で `Cache-Control` ヘッダを明示的に指定した場合のみキャッシュされます。|
|`FORCE_CACHE_ALL`|MIME タイプや `Cache-Control` ヘッダの設定に関わらず、強制的にすべてのコンテンツをキャッシュします。ユーザごとに出し分ける動的コンテンツなどもすべてキャッシュされてしまうので、GCS バックエンドなど、パブリックに公開して問題ないコンテンツを提供する場合に限って利用します。|


今回の GCS バックエンドの場合で考えると、例えば GCS バケットのすべてのオブジェクトをキャッシュ対象にしたい場合には、`FORCE_CACHE_ALL` に設定すると、確実にすべてのオブジェクトがキャッシュ対象となります。


### Default TTL / Max TTL / Client TTL とは

Cloud CDN では、3種類の TTL 値を使って、CDN におけるキャッシュの生存期間を設定します。
どの値が有効に効いてるくるかは、以下のようないくつかの条件で決まってきます。

- キャッシュモードで何が指定されているか？
- Origin が返す `Cache-Control` の設定値（`max-age`, `s-maxage`, `Expires`）

詳細な条件分岐は[公式ドキュメントに記載](https://cloud.google.com/cdn/docs/using-ttl-overrides)されているのでご確認ください。


#### 例：FORCE_CACHE_ALL の場合の挙動

ここでは、すべての条件を列挙しても仕方ないので、GCS バックエンドを使う場合によくある設定、つまり、キャッシュモードに `FORCE_CACHE_ALL` を指定して、すべてのコンテンツをキャッシュ対象にしているケースで考えてみます。

この場合、まず挙動に関係してくるのは `Default TTL` の値です。

> キャッシュ モードが FORCE_CACHE_ALL の場合、デフォルトの TTL は、すべてのレスポンスに設定されている TTL（送信元のヘッダーで TTL が設定されているレスポンスを含む）を上書きします。

https://cloud.google.com/cdn/docs/using-ttl-overrides?hl=ja#default-ttl


加えて、`Default TTL` より短い `Client TTL` を設定した場合には、`Client TTL` によってクライアント側でのキャッシュ期間を CDN 側のキャッシュ期間と別に設定できます。

（逆に言えば、`FORCE_CACHE_ALL` において、`Client TTL` >= `Default TTL` の場合は `Client TTL` の値は意味がなく、クライアントがローカルで持つキャッシュの有効期限と、CDN 側で持つキャッシュの有効期限がどちらも `Default TTL` の秒数に強制的に上書き設定されます。）

`Client TTL` が長いと、例えば CDN 側のキャッシュ期限が切れる直前にクライアントからアクセスがあった場合、CDN 側のキャッシュがその後更新されても、クライアント側キャッシュが期限切れになるまでは、古いコンテンツがローカルキャッシュから返ってしまいます。`Client TTL` を短く設定しておくと、ローカルのキャッシュ期限はすぐに切れるため、CDN 側のキャッシュの更新を頻繁にチェックする挙動となります。

`Max TTL` については `FORCE_CACHE_ALL` では設定できません。

> FORCE_CACHE_ALL では、TTL は常にデフォルトの TTL に設定されます。最大 TTL の設定はできません。

https://cloud.google.com/cdn/docs/using-ttl-overrides?hl=ja#max-ttl


### Serve stale content (Stale-while-revalidate) の設定

TTL の設定でもお腹いっぱいになりそうですが、Cloud CDN のキャッシュに関してもう一つ重要な設定として、[古いコンテンツを配信する（Serve stale content = いわゆる Stale-while-revalidate）機能](https://cloud.google.com/cdn/docs/serving-stale-content?hl=ja) の設定があります。

この機能を有効にした場合、CDN 側でのキャッシュ期限が切れていたとしても、指定した秒数だけは猶予期間として期限切れのキャッシュを CDN から返します。
Cloud CDN 側では、期限切れコンテンツへのリクエストがあってはじめて、Origin 側に非同期で最新のコンテンツを確認（Revalidate）しに行きます（逆に言うとリクエストがないと、`Serve stale content` の秒数が経過するまでずっと古いキャッシュが返ります）。

この機能がうれしいのは、以下のような理由です。

1. ユーザにエラーを表示する頻度を下げられる（Origin 再検証のタイミングでたまたま Origin がエラーを返しても、期限切れコンテンツを返してお茶を濁すことができる）
2. CDN でキャッシュを再生成する際のレイテンシが大きくなるのを防ぐことができる

特に 1. については、クラウドサービスの常として、GCS も低確率でエラー応答が返ることは想定する必要があるため、それをカバーできるのは便利だと思います。

またここでも重要な点としては、Cloud CDN で **明示的に指定しなかった場合には 86,400 秒 (1 日) がデフォルト値として勝手に設定される** ようになっているため、その間は CDN 側のキャッシュ期限が切れていても、期限切れのキャッシュがクライアントに返る可能性があります。

つまり極端に言えばデフォルト設定の場合、GCS 上でオブジェクトを更新した際に、CDN のキャッシュを 0 秒に指定していたとしても、最大 1 日後までは古いコンテンツがクライアントに返る可能性があります。

オブジェクトを更新した際に、一定期間内に確実にクライアント側に反映したい場合には、この `Serve stale content` も明示的に指定するようにしましょう。
Origin の `Cache-Control` ヘッダに `stale-while-revalidate` ディレクティブか、または Cloud CDN の `cdnPolicy.serveWhileStale` で明示的に指定を行うことができます。

<!-- ある時点で生成された Cloud CDN のキャッシュは、最長で `Default TTL` + `Serve while stale` の合計秒数の間利用される可能性あります。 -->


## じゃあどう設定すれば良いの？

ここまでの話を踏まえて、では GCS + HTTP(S) LB + Cloud CDN で静的コンテンツを配信する際に、各設定値はどのようにすれば良いか考えてみます。

まず以下の 2 点はある程度決め打ちできます。

- Cloud CDN でキャッシュするので、GCS ではオブジェクトの `Cache Metadata` に `no-store, max-age=0` を指定して組み込みキャッシュを無効化する
- GCS に動的コンテンツはないので、Cloud CDN のキャッシュモードは `FORCE_CACHE_ALL` を指定して、すべてのコンテンツをキャッシュ対象にする

あとは、Client TTL / Default TTL / Serve stale content の各秒数をいくつにすべきか、という点ですが、こちらは個別の要件に合わせて決めていくことになります。
基本的にはすべて短くした方が（0 秒に近づけた方が）オブジェクトを上書き更新した際にクライアントに反映されるタイムラグは短くなりますが、一方で以下のデメリットがあります。

- `Client TTL` を短くする: CDN へのリクエスト数が増える
- `Default TTL` を短くする: Origin へのリクエスト数が増える
- `Serve stale content` を短くする: クライアントへのエラー応答の確率が上がる。Origin まで見に行くレイテンシの長いリクエストが増える

メリット・デメリットを考慮して、秒数を決めるようにしましょう。


## 各タイミングで参照されるキャッシュの例

では改めて、ここまでご説明した内容を踏まえて、具体例を挙げて Client & CDN でのキャッシュの残り方を説明します。
GCS + HTTP(S) LB + Cloud CDN で静的コンテンツを配信する際を想定しています。
（ただし具体的な秒数はあくまで例です）。

**GCS の設定**

|設定項目|設定値|設定の意図|
|--|--|--|
|Cache Metadata|no-store, max-age=0|CDN を利用するので、GCS の組み込みキャッシュは利用しない|

**Cloud CDN の設定**

|設定項目|設定値|設定の意図|
|--|--|--|
|Cache Mode|FORCE_CACHE_ALL|すべてのコンテンツを強制的に CDN でキャッシュする|
|Client TTL|30 sec|CDN へのリクエスト数を減らすため、クライアント（ローカル）でも短時間キャッシュする|
|Default TTL|60 sec|CDN 側では、クライアントより長期間キャッシュする|
|Serve stale content|20 sec|Origin の一時的なエラーなどに備え、CDN 側キャッシュの期限切れ後も、ごく短時間の間は古いコンテンツの配信を許容する|

上記の設定において、各タイミングでクライアントからのリクエストが参照するキャッシュをまとめると、このようになります。

**各タイミングで参照されるキャッシュ**

![](/images/gcs-cloudcdn-cache/cache-diagram.png)

図を見ると、初回のリクエストで生成された CDN 側のキャッシュは、最終的に `Default TTL` + `Serve stale content` の合計秒数が経過すると、確実に更新されるのがわかります。


## まとめ

最後までお読みいただきありがとうございます。
Cloud CDN を中心に、いろいろな設定があってややこしかったと思いますが、なんとなくイメージはつかめたでしょうか？

SSG を使って事前レンダリングした Web サイトなど、このように GCS を使ってサクッと配信できると便利なケースは増えてきていると思います。
ぜひ本記事も参考にして、快適な GCS ライフを送っていただけると幸いです！