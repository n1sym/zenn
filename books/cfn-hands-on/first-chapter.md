---
title: "はじめに"
---

# はじめに

CloudFormationを使って、インフラ構築を自動化/コード化してみます。

ざっくりとした概要図ですが、完成系はこんな感じになります。

![](https://storage.googleapis.com/zenn-user-upload/0a76d8790ba7-20220516.png)

13章では Private Subnet に ECS を置くパターンも扱います。

# やらないこと

- ドメインの取得（HTTPS化）
- ECS の Auto Scaling
- CloudFormation の基本的な情報についての解説
  - 以下がとても参考になると思います
  - https://zenn.dev/soshimiyamoto/articles/3c624728438902

# 留意点

それぞれのリソースには依存関係があるので、この本は順番に進めていく必要があります。

![](https://storage.googleapis.com/zenn-user-upload/7836e70f465e-20220521.png)

また、赤背景になっている RDS, ECS のリソースは、起動しているだけで料金が発生するリソースです。今回作成する環境だと、1日あたり **3$ほどの料金が発生します** 。すぐ削除してしまえば少額に収まりますが、うっかり放置してしまうと大変なのでご注意ください。

# 事前準備

開発する PC にて、下記が利用できる / 済んでいる前提で進めます。

- git
- GitHub のユーザー登録
- Docker
- AWS のユーザー登録
- AWS CLI v2
  - 以下からインストールできます
  - https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html

# サンプルコード

https://github.com/n1sym/rails-cfn-hands-on