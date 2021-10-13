---
title: "Rails+AzureADでSAML認証のSSOを実装する"
emoji: "🔒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [rails, saml, azure, azuread, activedirectory]
published: true
---

検索しても詳しいやり方が出てこなかったので、備忘録を兼ねて。

# 前提

- Azureのアカウントを所持していること
  - 無料版の個人アカウントでもok
- Azure ADのテナントが用意されていること
  - https://docs.microsoft.com/ja-jp/azure/active-directory/develop/quickstart-create-new-tenant

# Railsアプリの前準備

## gem のインストール

SAMLのライブラリとして`ruby-saml` を、認証情報を扱うので`dotenv-rails`をインストール。

~~~ruby:Gemfile
gem 'ruby-saml'
gem 'dotenv-rails'
~~~

## 認証周りのコードを書く

`ruby-saml`はRails用の[サンプルコード](https://github.com/onelogin/ruby-saml-example)が用意されているので、それをそのままコピペする。

https://github.com/hukurouo/rails-sso/commit/0eb215529873b4fc7cc4a52dae00b726fb3a9318

（↑のコミットログではAccountテーブルを作っているが、実際は不要だった）


# Azure ADでSAMLの設定

https://portal.azure.com/#home

「Azure Active Directory」->「エンタープライズアプリケーション」->「新しいアプリケーション」->「独自のアプリケーションの作成」と辿る。

![](https://i.imgur.com/1QoGfPa.png)

任意のアプリ名を入力、ラジオボタンは一番下を選択して作成する。

アプリケーションが作成されたら、「シングルサインオンの設定」->「SAML」を選択し、SAMLによるシングルサインオンのセットアップに入ります。

## 基本的なSAML構成

ここでは以下の2点を入力する必要がある。

- 識別子（エンティティIID）
- 応答URL（Assertion Consumer Service URL）

これらの値は先ほどコピペしたRails側のコード（`app/models/account.rb`）でも設定されており、識別子が`settings.issuer`、応答URLが`settings.assertion_consumer_service_url`に対応している。


~~~ruby:app/models/account.rb
    #SP section
    settings.issuer                         = url_base + "/saml/metadata"
    settings.assertion_consumer_service_url = url_base + "/saml/acs"
~~~

今回はlocalhostから確認できればよいので、以下のように入力しておく。

- 識別子 (エンティティ ID)
  - `http://localhost:3000/saml/metadata`
- 応答 URL (Assertion Consumer Service URL)
  - `http://localhost:3000/saml/acs`

## ユーザー属性とクレーム

とりあえず最小限に、メールアドレスと名前のみを要求する。

![](https://i.imgur.com/cxP0V17.png)

一意のユーザー識別子は`user.mail`と設定。その横に書いてある`nameid-format:emailAddress`みたいなコードも確認しておく。(Railsアプリ側にも登録しておく必要があるため)


~~~ruby:app/models/account.rb
    settings.name_identifier_format = ENV['NAME_ID_FORMAT']
~~~

上記の例だと、`NAME_ID_FORMAT`はこのようなコードになる（最後を合わせてあげれば多分OK）

~~~
NAME_ID_FORMAT=urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress
~~~

## SAML署名証明書

ここでBase64証明書をダウンロードして、Railsアプリの`config`下に保存する。


~~~ruby:app/models/account.rb
    settings.idp_cert = File.read(Rails.root.join('config', 'rails-sso.cer'))
~~~

証明書の有効期限が切れたら作り直す必要があるので留意しておきましょう。


## (任意のアプリ名)のセットアップ

AzureAD識別子、ログインURL、ログアウトURLが表示されているので、Railsアプリに登録する。SSOがシングルサインオン、SLOがシングルログアウトの略。


~~~ruby:app/models/account.rb
    # IdP section
    settings.idp_entity_id = ENV['IDP_ENTITY_ID']    # AzureAD識別子
    settings.idp_sso_target_url = ENV['IDP_SSO_URL'] # ログインURL
    settings.idp_slo_target_url = ENV['IDP_SLO_URL'] # ログアウトURL
~~~

最後に「ユーザーとグループ」で自分のアカウントを割り当てて、Azure ADでの設定は完了。

# RailsでSSO

`http://localhost:3000`を開くとLoginリンクが表示される。

上手くいっていたらSAML認証のSSOが実行されるはず。

# リポジトリ

https://github.com/hukurouo/rails-sso

