---
title: "API Gatewayを使う"
---


API GatewayでREST APIを作成して、Lambda関数を繋げてAPIを完成させます。

REST API を選択します。

![](https://storage.googleapis.com/zenn-user-upload/rc9uldzt8oe2msaky03etsrxre1h)

新しいAPI、API名は任意、他はデフォルト設定で、APIの作成をクリックします。

![](https://storage.googleapis.com/zenn-user-upload/skzshczf4lvbh7lygc41eulty7w9)

アクション▼からリソースの作成を選びます。

![](https://storage.googleapis.com/zenn-user-upload/yrx1ueccxixvmzdumg2l89i1kh5w)

リソース名は任意、API Gateway CORS を有効にするにチェックを入れてリソースの作成をクリックします。

![](https://storage.googleapis.com/zenn-user-upload/q2fc6sxme9tsje0zb70g6pmv0bj7)

アクション▼からメソッドの作成、セレクトメニューでGETを選びます。

統合タイプは Lambda 関数、Lambda プロキシ統合の使用にチェックを入れて、Lambda 関数はさきほど作った関数名を入力して、保存をクリックします。


![](https://storage.googleapis.com/zenn-user-upload/5p3r8ovj2k2uwexzsq2p3tfxiwvv)

「API GatewayからLambda関数を呼び出す権限を与える」旨のメッセージが表示されるので、［OK］ ボタンをクリックします。

アクション▼からAPIのデプロイを選びます。各項目は適当に入力して、デプロイします。

![](https://storage.googleapis.com/zenn-user-upload/dfe73z9smq8ley2rowyfyujbgci4)

APIのエンドポイントが表示されるので、コピーしておきます。

試しにアクセスしてみます。

https://*********.execute-api.ap-northeast-1.amazonaws.com/production/fav?name=hukurouo

`?name=hukurouo` のところがパラメータになっていて、lambda関数では`event.queryStringParameters.name`でそれを受け取っています。

~~~js
const params = { screen_name: event.queryStringParameters.name, count: 180 };
~~~