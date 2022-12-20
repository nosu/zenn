---
title: "GCS + LB + Cloud CDN で静的コンテンツを配信する際のキャッシュについておさらいする"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

本記事は [Google Cloud Japan Advent Calendar 2022](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370) の [通常版](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370#%E9%80%9A%E5%B8%B8%E7%89%88) の 19 日目の記事です。

Google Cloud Storage (GCS) では、Web サイトなどの静的コンテンツを、バケットに保存するだけで簡単に公開することができて便利です。
さらに、HTTP(S) Load Balancing と組み合わせることで、HTTPS に対応させたり、Cloud CDN と組み合わせることで CDN にキャッシュすることもできるので、アクセス数の多い商用サービスなどでも充分に対応できます。

一方で、このやり方を採用してシステム設計する際に、ややこしいのがキャッシュの設定です。
特に上記のように LB や CDN と組み合わせた際には、以下のようにキャッシュに関連する設定が複数あるため、挙動が少しわかりにくいです。

- GCS の組み込みキャッシュ設定
- Cloud CDN の TTL 設定
- Cloud CDN の Serve while stale (Stale-while-revalidate) 設定

本記事では、それぞれの設定がどのような挙動になるのか説明します。


## 先にまとめ

GCS + Cloud CDN で静的コンテンツを配信する際の設定パターンの一例をあげると、例えば以下のようなケースがあります
（具体的な秒数はあくまで例です）。

**GCS の設定**

|設定項目|設定値|設定の意図|
|--|--|--|
|Cache Metadata|no-store, max-age=0|CDN を利用するので、GCS の組み込みキャッシュは利用しない|

**Cloud CDN の設定**

|設定項目|設定値|設定の意図|
|--|--|--|
|Cache Mode|FORCE_CACHE_ALL|すべてのコンテンツを強制的にキャッシュする|
|Client TTL|30 sec|クライアント（ローカル）でも短時間キャッシュする|
|Default TTL|60 sec|CDN 側では、クライアントより長期間キャッシュする|
|Serve stale content|20 sec|Origin の一時的なエラーなどに備え、CDN 側キャッシュの期限切れ後も、ごく短時間の間は古いコンテンツの配信を許容する|


上記の設定において、各タイミングでクライアントからのリクエストが参照するキャッシュをまとめると、このようになります。

**各タイミングで参照されるキャッシュ**

![](/images/gcs-cloudcdn-cache/cache-diagram.png)

こう見ると、やはりなかなかややこしく見えてしまいますね。
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
その場合、HTTP(S) Load Balancing で Cloud CDN との連携機能を有効にすることで、CDN 経由で低レイテンシでのコンテンツ配信を行うことができます。

Cloud CDN - HTTP(S) LB - GCS という構成にする場合、以下のような設定がキャッシュの挙動に関わってきます。

- キャッシュモード（`CACHE_ALL_STATIC` / `USE_ORIGIN_HEADERS` / `FORCE_CACHE_ALL` のいずれか）
- 各種 TTL 設定（`Default TTL` / `Max TTL` / `Client TTL` のそれぞれ）
- Serve stale content (`Stale-while-revalidate`) の設定

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

例として、GCS バックエンドを使っていて、キャッシュモードに `FORCE_CACHE_ALL` を指定することで、すべてのコンテンツをキャッシュ対象にしている場合で考えてみます。

この場合、まず挙動に関係してくるのは `Default TTL` の値です。

> キャッシュ モードが FORCE_CACHE_ALL の場合、デフォルトの TTL は、すべてのレスポンスに設定されている TTL（送信元のヘッダーで TTL が設定されているレスポンスを含む）を上書きします。

https://cloud.google.com/cdn/docs/using-ttl-overrides?hl=ja#default-ttl


加えて、`Default TTL` より短い `Client TTL` を設定した場合には、`Client TTL` によってクライアント側でのキャッシュ期間を CDN 側のキャッシュ期間と別に設定できます。


逆に言えば、`FORCE_CACHE_ALL` において、`Client TTL` >= `Default TTL` の場合は `Client TTL` の値は意味がなく、クライアントがローカルで持つキャッシュの有効期限と、CDN 側で持つキャッシュの有効期限がどちらも `Default TTL` の秒数に強制的に上書き設定されます。

`Client TTL` が長いと、例えば CDN 側のキャッシュ期限が切れる直前にクライアントからアクセスがあった場合、CDN 側のキャッシュがその後更新されても、クライアント側キャッシュが期限切れになるまでは、古いコンテンツがローカルキャッシュから返ってしまいます。`Client TTL` を短く設定しておくと、ローカルのキャッシュ期限はすぐに切れるため、CDN 側のキャッシュの更新を頻繁にチェックする挙動となります。

`Max TTL` については `FORCE_CACHE_ALL` では設定できません。

> FORCE_CACHE_ALL では、TTL は常にデフォルトの TTL に設定されます。最大 TTL の設定はできません。

https://cloud.google.com/cdn/docs/using-ttl-overrides?hl=ja#max-ttl


## Serve stale content (Stale-while-revalidate) の設定

TTL の設定でもお腹いっぱいになりそうですが、Cloud CDN のキャッシュに関してもう一つ重要な設定として、`Serve stale content`（古いコンテンツを配信する）機能の設定があります。

Client TTL = 60


60 秒後まで：Client 側のキャッシュが返る


https://cloud.google.com/cdn/docs/serving-stale-content?hl=ja




## おまけ

GCS を HTTP(S) LB の配下に置く場合に、GCS への直接アクセスは禁止したい！というケースは多いかと思います。
その場合どうすれば良いかは、以下に素晴らしい記事があるのでこちらを参照ください。

[Google Cloud Storage をロードバランサーのバックエンドにしつつ、直接はアクセスさせたくない場合](https://medium.com/google-cloud-jp/private-cloud-storage-bucket-for-load-balancing-3975c1d2b743)

