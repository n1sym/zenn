---
title: "Twitter API の準備"
---

# Twitter API の利用申請をする

初めて Twitter API を使用する場合、審査を受けて許可をもらう必要があります。

英語で利用目的などを書く必要がありますが、機械翻訳でも全然大丈夫でした。申請手順は以下が参考になると思います。

https://dev.classmethod.jp/articles/twitter-api-approved-way/

審査は1~2日で終わるみたいです。自分は12時間後くらいでした。

# APIキーとアクセストークンを取得する

`developer.twitter.com` は最近ページ仕様が変わったので、ここは画像付きで説明していきます。

https://developer.twitter.com/en/portal/apps/new

上記リンクから新規アプリを作ります。名前は任意で大丈夫です。

![](https://storage.googleapis.com/zenn-user-upload/6j4xbsyqj4875wttrc11dbjyzsid)

作成に成功したら`API key`, `API key secret` が表示されるので、保存しておきます。`Bearer token`については今回は不要なので、スルーします。

![](https://storage.googleapis.com/zenn-user-upload/e93jezt28ert6auzqknfv8e6hz2c)

App settings に進んで、ナビゲーションバーから Keys and tokens に遷移します。

![](https://storage.googleapis.com/zenn-user-upload/9mm02rwdbggxnm2fax9yp699cq81)

Authentication Tokens の項目にある Access token & secret の generate ボタンを押下し、`access_token_key`, `access_token_secret`を取得して、保存します。

これにて準備完了です。



