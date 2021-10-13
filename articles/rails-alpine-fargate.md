---
title: "alpineベースのRailsをFargateにデプロイする"
emoji: "🚚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [rails, alpine, fargate]
published: true
---

前回作ったRailsアプリをAWS Fargateにデプロイする。

https://zenn.dev/hukurouo/articles/rails-alpine

DBの接続テストもしたいので、適当にユーザーモデルを生成しておく。

~~~
$ docker exec -it alpine-sandbox_web_1 bash 
$ rails generate scaffold User name:string email:string
$ rails db:migrate
~~~

# ロールの用意

Fargateのデプロイには2種のロールが必要なので準備する。

### ecsTaskExecutionRole

ECSのタスクを実行するためのロール。以下のAWS管理ポリシーをアタッチする。

- AmazonECSTaskExecutionRolePolicy
  - デフォのポリシー
- AmazonSSMReadOnlyAccess
  - System Managerに秘匿した環境変数を読む用

加えて、ECS Exec用のポリシーも新規に作ってアタッチしておく。

~~~json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssmmessages:CreateControlChannel",
        "ssmmessages:CreateDataChannel",
        "ssmmessages:OpenControlChannel",
        "ssmmessages:OpenDataChannel"
      ],
      "Resource": "*"
    }
  ]
}
~~~

https://dev.classmethod.jp/articles/ecs-exec/#toc-7

### AWSCodeDeployRole

BGデプロイを使用する場合に必要になるロール。

AWS 管理ポリシー AWSCodeDeployRoleForECS をアタッチする。

サービスロール作成 => ユースケースの選択 => CodeDeploy - ECS で自動でアタッチされる。


# AWSコンソールでの設定

以下の記事を参考に各項目を設定する。

https://qiita.com/hatsu/items/22e11e94a0a981d78efa

ひとつでもミスると全滅するので慎重に･･･。

# Rails側の設定

以下は↑の記事の通りに設定すればOK

- `database.yml`の編集
- HealthCheck用のAPIを作成
- `.dockerignore`の作成
- `nginx.conf`の作成
- `config/puma.rb`の編集

alpine用にカスタマイズが必要なところを修正していく。

~~~Dockerfile:docker/production/Dockerfile
FROM ruby:3.0.2-alpine

ENV RAILS_ENV production
ENV BUNDLE_WITHOUT development:test

WORKDIR /app
COPY Gemfile Gemfile.lock package.json yarn.lock /app/

RUN apk add --no-cache -t .build-dependencies \
    alpine-sdk \
    build-base \
 && apk add --no-cache \
    bash \
    mysql-dev \
    nodejs \
    tzdata \
    yarn \
    npm \
    nginx \
    openrc \
 && gem install bundler:2.0.2 \
 && bundle install \
 && yarn install --production --frozen-lockfile && yarn cache clean \
 && apk del --purge .build-dependencies

COPY . /app

RUN SECRET_KEY_BASE=placeholder bundle exec rails assets:precompile \
 && yarn cache clean \
 && rm -rf node_modules tmp/cache

# nginx
ADD docker/production/nginx.conf /etc/nginx/nginx.conf
ADD docker/production/entrypoint.sh /app/entrypoint.sh

# openrc preparation
# ref: https://stackoverflow.com/questions/65627946
RUN openrc && touch /run/openrc/softlevel

EXPOSE 80

RUN chmod +x /app/entrypoint.sh
~~~

alpineイメージは`service`が使えないので、代わりに`openrc`を使用した。

https://wiki.alpinelinux.org/wiki/Alpine_Linux_Init_System


~~~bash:docker/production/entrypoint.sh
#!/bin/bash

# ref: https://github.com/gliderlabs/docker-alpine/issues/42#issuecomment-173825611
# Tell openrc loopback and net are already there, since docker handles the networking
echo 'rc_provide="loopback net"' >> /etc/rc.conf
rc-service nginx start

cd /app
RAILS_ENV=production bin/rails db:migrate
bundle exec pumactl start
~~~

# デプロイ

プッシュコマンドはECRのコンソール画面で確認できる。

buildコマンドは本番環境用の`Dockerfile`をオプションで指定する必要あり。
~~~
docker build -f docker/production/Dockerfile -t rails-alpine-sandbox .
~~~

2~3分くらい経ったらサーバーが立ち上がるので、ALBのDNS名にアクセス。動いたら成功。

動かなかったら･･･Cloudwatchに吐かれているログをチェック、そもそもタスクが立ち上がっていない場合は失敗したタスクの詳細ページに失敗理由が書かれているので確認してみるとよいかも。

# ECS Exec でコンテナに接続する

https://dev.classmethod.jp/articles/ecs-exec/

の通りにやればよい。bashでログインもできるのでかなり便利。

# リポジトリ

https://github.com/hukurouo/rails-alpine-sandbox