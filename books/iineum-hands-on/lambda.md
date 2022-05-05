---
title: "AWS Lambdaを使う"
---

# CORSについて

TwitterAPIが使えるようになりましたが、Reactアプリから（ブラウザから）直接呼び出そうとするとCORSエラーで弾かれてしまいます。

参考：
https://developer.mozilla.org/ja/docs/Web/HTTP/CORS

つまり、新規にWEB APIサーバーを作る必要があります。今回はAWS Lambda + API Gateway を使って、APIサーバーを作っていきます。

# Lambdaのセットアップ

Lambda関数を作ります。一から作成で、関数名は任意、ランタイムは`Node.js 12.x`を選びます。他はデフォルト設定のままで大丈夫です。

![](https://storage.googleapis.com/zenn-user-upload/ata78za40u01bcnlrl82xrgjcz0m)

今回は外部の[ライブラリ](https://www.npmjs.com/package/twitter)を使用するので、関数コードをzipファイルでアップロードする必要があります。

任意のディレクトリを作って、コードを作成します。

~~~sh
$ mkdir twitter-iine-getter
$ cd twitter-iine-getter
$ npm init -f
$ touch index.js
$ npm install twitter --save
~~~

とりあえずで、`index.js`にはAPIとの疎通が確認できる最低限のコードを書いておきます。

~~~js:index.js
const twitter = require('twitter');
 
const client = new twitter({
  consumer_key: process.env.consumer_key,
  consumer_secret: process.env.consumer_secret,
  access_token_key: process.env.access_token_key,
  access_token_secret: process.env.access_token_secret
});

exports.handler = async () => {
  const params = { screen_name: "hukurouo", count: 1 };
  await client
    .get("favorites/list", params)
    .then((tweet) => {
      console.log(tweet);
    })
    .catch((error) => {
      throw error;
    });
  return {
    statusCode: 200,
    body: "success!"
  };
};
~~~

パラメータは書き換えてもらっても大丈夫です。自分のツイッターIDにしておくと結果が分かりやすいかも？ `count`については最大値が200なのでご留意ください。

作ったファイルを圧縮します。自分はwindows環境だったのでファイルエクスプローラから直接圧縮しました。

~~~sh
$ zip -r lambdaupload.zip ./index.js ./node_modules/
~~~

.zipファイルをアップロードします。

![](https://storage.googleapis.com/zenn-user-upload/jxbf8adtf6obr0sxna3umplymi9m)

# 環境変数の設定

TwitterAPIの各種キーを環境変数として設定します。ちょっとややこしいのですが、`consumer_key`は`API key`の値を、`consumer_secret`には`API key secret`の値を入力してください。

![](https://storage.googleapis.com/zenn-user-upload/zy7p4dg5aqvwssyvbhb05bk4lyof)

これで先ほどのコードが実行できるようになります。右上のテストボタンからテストを実行してみましょう。テストイベントは初期値に適当に名前をつけたもので大丈夫です。

![](https://storage.googleapis.com/zenn-user-upload/7wgc977bftwntw66q6nj28p6rfgp)

