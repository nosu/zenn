---
title: "Firestore を使って、サーバサイドから Web ブラウザにリアルタイムに更新を通知する"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gcp, cloudstorage, cloudcdn, cloudloadbalancer]
published: false
---

本記事は [Google Cloud Japan Advent Calendar 2022](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370) の [通常版](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370#%E9%80%9A%E5%B8%B8%E7%89%88) の 19 日目の記事です。


## Firestore のリアルタイムリスナ機能とは？

Firestore は、Google Cloud が提供する、NoSQL のデータベースサービスです。
Firestore の全体像や基本的な使い方については、[101 版の Advent Calendar]() で、素晴らしい入門記事が投稿されているので、そちらをぜひお読みください。

本記事でご紹介するのは、この Firestore の非常に強力な機能として、リアルタイムリスナという機能があります。
リアルタイムリスナとは、クライアントアプリから Firebase 上のドキュメントを "リッスン" することで、Firestore 側に何かしらのデータ更新が入ったときに、（ほぼ）リアルタイムに最新のデータを受け取って任意の処理を実行できる仕組みです。

Google Cloud を使っている方でも、普段フロントエンドよりサーバサイド中心に開発されていると、意外と知らない

この機能を使うと、非常に少量のコード実装で、リアルタイムに通知を

例えば、ーーーのユースケースで考えてみましょう。

作成処理には少し時間がかかるので、作成リクエストを受けたサーバアプリは、非同期で別のジョブを実行して作成処理を行っているとします。
さて、このバッチ処理が完了した際に、エンドユーザの見ているブラウザ上に完了通知を表示するためにはどうすれば良いでしょうか？

Firestore を利用すると、非常に簡単に通知を行うことができます。


## リアルタイムリスナを使ってみる

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Firestore Realtime Listener Sample</title>
</head>
<body>
  <div id="main"></div>

  <script src="https://www.gstatic.com/firebasejs/9.14.0/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.14.0/firebase-firestore-compat.js"></script>
  <script src="index.js"></script>
</body>
</html>
```

```javascript
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};

firebase.initializeApp(firebaseConfig);

const db = firebase.firestore();
const doc = db.collection('users').doc('user01');
console.log("doc", doc);

const observer = doc.onSnapshot(docSnapshot => {
  const data = docSnapshot.data();
  console.log("Doc has been updated.", data);
}, err => {
  console.log(`Encountered error: ${err}`);
});
```





- 





|種類|説明|
|----|----|
|ポーリング|一定の間隔を空けながらクライアントから更新を取得しに行く|
|ロングポーリング||
|Websocket||
|gRPC 双方向ストリーミング||

Firestore を利用すると、実装を