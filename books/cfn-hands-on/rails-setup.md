---
title: "Rails アプリのセットアップ"
---

まずは Rails アプリを Docker でセットアップします。

# 前準備

公式チュートリアルを参考に、必要なファイルを揃えます。

https://docs.docker.com/samples/rails/


~~~ruby:Gemfile
source 'https://rubygems.org'
gem 'rails', '~> 7.0', '>= 7.0.3'
~~~

Rails は記事執筆時点で最新バージョンである [v7.0.3](https://rubygems.org/gems/rails/versions/7.0.3) を使います。

`Gemfile.lock` は空ファイルを作成。
~~~
touch Gemfile.lock
~~~

`entrypoint.sh` はチュートリアルのものをそのまま使用。

~~~sh:entrypoint.sh
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /myapp/tmp/pids/server.pid

# Then exec the container's main process (what's set as CMD in the Dockerfile).
exec "$@" 
~~~

続いて Dockerfile を用意します。Ruby のイメージはできるだけ[最新のもの](https://hub.docker.com/_/ruby)を使います。 

~~~Dockerfile:Dockerfile
FROM ruby:3.1.2-alpine3.15

WORKDIR /myapp
COPY Gemfile Gemfile.lock /myapp/

RUN apk add --no-cache -t .build-dependencies \
    alpine-sdk \
    build-base \
 && apk add --no-cache \ 
    postgresql-dev \ 
    bash \
    tzdata \
 && gem install bundler \
 && bundle install \
 && apk del --purge .build-dependencies

COPY . /myapp

# Add a script to be executed every time the container starts.
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

# Configure the main process to run when running the image
CMD ["rails", "server", "-b", "0.0.0.0"]
~~~

イメージは軽いほど良いので、alpineイメージをベースにして必要なライブラリのみをインストールします。Rails 7 になってから Node.js が必要なくなったので大分すっきりしましたね。

ビルドにしか使わないライブラリは `.build-dependencies` タグをつけておいて、最後に消去 `apk del --purge` しています。

最後は docker-compose.yml ファイルです。データベースは `postgres` を使います。

~~~yml:docker-compose.yml
version: "3.9"
services:
  db:
    image: postgres:14-alpine
    restart: always
    environment:
      POSTGRES_PASSWORD: example
      TZ: "Asia/Tokyo"
    volumes:
      - postgres:/var/lib/postgresql/data
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db
    environment:
      - DATABASE_HOST=db
      - DATABASE_USERNAME=postgres
      - DATABASE_PASSWORD=example
    logging:
      driver: json-file
      options:
        max-file: '1'
        max-size: 3m
volumes:
  postgres:
    driver: local
~~~

docker 環境は何も設定しないと無限にログが溜まっていくので、`logging:` で ログローテーションの設定をしています。

これで準備は完了です。現時点でディレクトリは以下のようになっています。

~~~
.
├── docker-compose.yml
├── Dockerfile
├── entrypoint.sh
├── Gemfile
└── Gemfile.lock
~~~

# ビルド

`rails new` を実行して、Railsアプリの骨組みを作ります。

~~~sh:setup.sh
apk add git alpine-sdk build-base
rails new . --force --database=postgresql
~~~

セットアップ用のシェルスクリプトを書いて実行します。（初期設定のときだけ必要なライブラリがあるので一時的にインストールしています）

~~~
docker-compose run --rm --no-deps web sh setup.sh
~~~

諸々のファイルが生成されたら、イメージをビルドします。

~~~
docker-compose build
~~~

# データベースの設定

続いてデータベースの設定を行います。

コンテナ環境で動いている postgres と接続するため `config/database.yml` に以下を追記します。

~~~diff yml:config/database.yml
 default: &default
   adapter: postgresql
   encoding: unicode
   # For details on connection pooling, see Rails configuration guide
   # https://guides.rubyonrails.org/configuring.html#database-pooling
   pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
+  username: <%= ENV['DATABASE_USERNAME'] %>
+  password: "<%= ENV['DATABASE_PASSWORD'] %>"
+  host: <%= ENV['DATABASE_HOST'] %>
~~~

今後の章で自動生成するデータベースのパスワードの最初の文字が `,` や `@` などになった場合に上手くパースされないので、`password:` の項目を `"` で囲っています。

（以下のようなエラーが出る）

> YAML syntax error occurred while parsing /app/config/database.yml. Please note that YAML must be consistently indented using spaces. Tabs are not allowed. Error: (<unknown>): did not find expected node content while parsing a block node at line 28 column 13

以下のコマンドを実行して、データベースを作成します。
~~~
docker-compose run --rm web rake db:create
~~~

# アプリの起動

これでアプリを起動する準備ができたので、コンテナを立ち上げます。

~~~
docker-compose up -d
~~~

`http://localhost:3000` に接続して以下の画面が表示されたら OK です。

![](https://storage.googleapis.com/zenn-user-upload/cdc516247c6b-20220521.png)

# scaffold で簡単なアプリケーションを生成

データベースとの接続も試しておきたいので scaffold ジェネレータを使って簡単なアプリケーションを生成します。

~~~
docker-compose run --rm web rails g scaffold Tweet title:string content:text
docker-compose run --rm web rails db:migrate
~~~

`config/routes.rb` で rootパスを書き換えておきます。
~~~diff rb:config/routes.rb
 Rails.application.routes.draw do
+  root "tweets#index"
   resources :tweets
   # Define your application routes per the DSL in https://guides. rubyonrails.org/routing.html
 
   # Defines the root path route ("/")
   # root "articles#index"
end
~~~

Twitter を模した簡単なアプリケーションが操作できるようになりました。

![](https://storage.googleapis.com/zenn-user-upload/d08fb16a1c60-20220423.png)

# デプロイ用の設定

AWS にデプロイする際に必要な設定もしておきます。デプロイに必要なファイルは `deploy/development/` 下に配置していきます。


~~~Dockerfile:deploy/development/Dockerfile
FROM ruby:3.1.2-alpine3.15

WORKDIR /app
COPY Gemfile Gemfile.lock /app/

RUN apk add --no-cache -t .build-dependencies \
    alpine-sdk \
    build-base \
 && apk add --no-cache \
    bash \
    postgresql-dev \ 
    tzdata \
    nginx \
    openrc \
 && bundle install \
 && apk del --purge .build-dependencies

COPY . /app

RUN SECRET_KEY_BASE=placeholder bundle exec rails assets:precompile

# nginx
ADD deploy/development/nginx.conf /etc/nginx/nginx.conf
ADD deploy/development/entrypoint.sh /app/entrypoint.sh

# openrc preparation
# ref: https://stackoverflow.com/questions/65627946
RUN openrc && touch /run/openrc/softlevel

EXPOSE 80

RUN chmod +x /app/entrypoint.sh 
~~~

alpine イメージは `service` が使えないので、代わりに `openrc` を使用しています。

https://wiki.alpinelinux.org/wiki/Alpine_Linux_Init_System

コンテナが立ち上がった時に起動するコマンドを以下のように設定します。nginx の起動、DB の migrate、Rails サーバーの立ち上げという流れになっています。

~~~sh:deploy/development/entrypoint.sh
#!/bin/bash

# ref: https://github.com/gliderlabs/docker-alpine/issues/42#issuecomment-173825611
# Tell openrc loopback and net are already there, since docker handles the networking
echo 'rc_provide="loopback net"' >> /etc/rc.conf
rc-service nginx start

cd /app

bundle exec rake db:migrate
bundle exec pumactl start 
~~~

nginx の設定ファイルも作成します。

~~~:deploy/development/nginx.conf
user nginx;
worker_processes auto;
error_log /dev/stderr info;
pid /var/log/nginx.pid;

events {
  worker_connections 1024;
}

http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;

  log_format json escape=json '{"time": "$time_iso8601",'
        '"remote_addr": "$remote_addr",'
        '"host": "$host",'
        '"http_host": "$http_host",'
        '"proxy_add_x_forwarded_for": "$proxy_add_x_forwarded_for",'
        '"scheme": "$scheme",'
        '"remote_user": "$remote_user",'
        '"status": "$status",'
        '"server_protocol": "$server_protocol",'
        '"request_method": "$request_method",'
        '"request_uri": "$request_uri",'
        '"request": "$request",'
        '"body_bytes_sent": "$body_bytes_sent",'
        '"bytes_sent": "$bytes_sent",'
        '"request_time": "$request_time",'
        '"upstream_response_time": "$upstream_response_time",'
        '"upstream_connect_time": "$upstream_connect_time",'
        '"upstream_addr": "$upstream_addr",'
        '"http_user_agent": "$http_user_agent",'
        '"http_referer": "$http_referer"}';

  access_log  /dev/stdout  json;

  sendfile        on;
  keepalive_timeout  65;

  upstream app {
    server unix:///app/tmp/sockets/puma.sock;
  }

  server {
    listen 80 default_server;

    root /app/public;

    location / {
      proxy_pass http://app/;
    }

    client_max_body_size 100m;
    keepalive_timeout 5;
  }
} 
~~~

nginx からのアクセスを許可しておきます。

~~~diff rb:config/environments/development.rb
   # config.action_cable.disable_request_forgery_protection = true
 
+  config.hosts << "app"
 end
~~~

Web サーバーは `puma` を使います。

~~~diff rb:config/puma.rb
  # Allow puma to be restarted by `bin/rails restart` command.
  plugin :tmp_restart

+ app_root = File.expand_path('..', __dir__)
+ bind "unix://#{app_root}/tmp/sockets/puma.sock" 
~~~

ヘルスチェック用の API も生やしておきます。

~~~rb:app/controllers/api/health_check_controller.rb
class Api::HealthCheckController < ApplicationController
  def index
    render json: {status: 200}, status: 200
  end
end
~~~

~~~diff rb:config/routes.rb
+  namespace :api do
+   resources :health_check, only: :index
+  end
~~~

CSRF 保護をオンにしておきます。

:::message
Rails の CSRF 対策については以下が参考になると思います。
https://railsguides.jp/security.html#%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%B5%E3%82%A4%E3%83%88%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88%E3%83%95%E3%82%A9%E3%83%BC%E3%82%B8%E3%82%A7%E3%83%AA%EF%BC%88csrf%EF%BC%89
:::



~~~diff rb:app/controllers/tweets_controller.rb
 class TweetsController < ApplicationController
   before_action :set_tweet, only: %i[ show edit update destroy ]

+  protect_from_forgery
~~~



`/tmp/sockets` ディレクトリを GitHub に置いておく必要があるので、`.gitignore` に以下を追記して、

~~~diff gitignore:.gitignore
  /tmp/*
  !/log/.keep
  !/tmp/.keep
+ !/tmp/sockets
~~~

`/tmp/sockets`下に `.keep` ファイルを置いておきます。 

~~~
...
├─ tmp
│   └─ sockets
│       └─ .keep
...
~~~

これで Rails アプリのセットアップは完了です！