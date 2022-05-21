---
title: "AWS CLI v2 のセットアップ"
---

コマンドラインインターフェイスから AWS リソースを操作したいので、AWS CLI v2 を使えるようにしておきます。

:::message
既に環境に AWS CLI v2 がインストールされている場合は、この章はスキップ可能です。
:::

# ファイルの準備

直接PCにインストールしてもよいのですが、せっかくなので docker で環境を作ります。

`aws-cli` というディレクトリを作り、そこに `Dockerfile` を用意します。

~~~Dockerfile:aws-cli/Dockerfile
FROM python:3.9

RUN apt-get update && apt-get install -y less vim curl unzip sudo \
 && curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
 && unzip awscliv2.zip \
 && sudo ./aws/install

WORKDIR /myapp
~~~

https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html

`docker-compose.yml` に以下を追記。

~~~diff yml:docker-compose.yml
       - DATABASE_HOST=db
       - DATABASE_USERNAME=postgres
       - DATABASE_PASSWORD=example
+  aws-cli-container:
+    build: ./aws-cli
+    container_name: awscli-container
+    volumes:
+      - .:/myapp
+    env_file:
+      - .env
+    environment:
+      AWS_DEFAULT_REGION: ap-northeast-1
+      AWS_DEFAULT_OUTPUT: json
 volumes:
   postgres:
     driver: local 
~~~

認証情報は `.env` で読み込み。アクセスキーはIAMユーザーの認証情報タブから取得できます。

~~~text:.env
AWS_ACCESS_KEY_ID=YOUR_AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY=YOUR_AWS_SECRET_ACCESS_KEY
~~~



:::message alert
**（要確認！）** `.gitignore` に `.env` を追加しておきます。認証情報は絶対に GitHub には公開しないようにしましょう。
:::

~~~diff gitignore:.gitignore
+ .env 
~~~

# ビルドして確認

イメージをビルドして、

~~~
docker-compose build 
~~~

試しに S3 のリストを取得してみます。

~~~
docker-compose run --rm aws-cli-container aws s3 ls
2022-04-17 08:51:32 cfn-sample-athena-results-test
2022-04-16 09:30:47 cfn-sample-logs-dev
~~~

無事取得できました。

また、以下のコマンドでコンテナに入り込んで、直接 `aws s3 ls` を実行することも可能です。
~~~
docker-compose run --rm aws-cli-container /bin/bash
~~~