---
title: "netlifyへデプロイ"
---

# デプロイの準備

netlifyで環境変数を使うため、以下をルートディレクトリに追加しておきます。

~~~toml:netlify.toml
[context.production.environment]
  TOML_ENV_VAR = "From netlify.toml"
  REACT_APP_TOML_ENV_VAR = "From netlify.toml (REACT_APP_)"
~~~

参考：
https://docs.netlify.com/configure-builds/file-based-configuration/#sample-file

# netlifyへデプロイ

https://www.netlify.com/

アカウントを持っていない人は、GitHubアカウントでの新規登録を行います。

ログインが済んだら「New site form Git」でデプロイ作業に移ります。

1. GitHubを選択し、認証を行います。

2. 作ったリポジトリを選択


![](https://storage.googleapis.com/zenn-user-upload/2hcb9pfsrannvj746vfgcy1znlzh)

Branch to deploy 欄は `master` の場合もあります（メインのブランチ名で変わる）

次に「Show advanced」を押したあとに表示される「New Variables」をクリックし、以下の環境変数を入力します。この値は`.env`に記載した値と同じものです。

| Key |	Value |
| ---- | ---- | 
| REACT_APP_API_ENDPOINT_URL | ご自身のAPIエンドポイント | 


Deploy site を押したら、あとはビルドとデプロイが完了するのを待ちます。

デプロイが完了すれば、URLが表示されます。

https://hopeful-jepsen-deda42.netlify.app/

無事にデプロイされました。