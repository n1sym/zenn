---
title: "GitHub Actions で docker build して ECR に push する"
emoji: "🏭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [githubactions, ecr]
published: true
---

ちょくちょく詰まりポイントがあったので、備忘録として。

# IAMリソースの作成

まず、GitHub Actions からAWSリソースを操作するための認証を行う必要があります。

最近追加された、GitHub Actions の OpenID Connect (OIDC) を使うのがよさそう。

> 参考: https://zenn.dev/miyajan/articles/github-actions-support-openid-connect

[公式テンプレート](https://github.com/aws-actions/configure-aws-credentials#sample-iam-role-cloudformation-template)を参考に、CloudFormation でIAMリソースを作成していきます。

~~~yml:cfn-iam.yml
AWSTemplateFormatVersion: "2010-09-09"
Description: IAM resource
Parameters:
  GitHubOrg:
    Type: String
    Default: hukurouo # GitHub のアカウント名
  RepositoryName:
    Type: String
    Default: github-actions-ecr-push-test # リポジトリ名 
  AccountId:
    Type: String

Resources:
  GitHubActionsRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: github-actions-role
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Action: sts:AssumeRoleWithWebIdentity
            Principal:
              Federated: !Ref GithubOidc
            Condition:
              StringLike:
                token.actions.githubusercontent.com:sub: !Sub repo:${GitHubOrg}/${RepositoryName}:*
      Policies:
        - PolicyName: ECRTestPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - ecr:GetAuthorizationToken
                Resource: '*'
              - Effect: Allow
                Action:
                  - ecr:GetDownloadUrlForLayer
                  - ecr:BatchGetImage
                  - ecr:InitiateLayerUpload
                  - ecr:PutImage
                  - ecr:UploadLayerPart
                  - ecr:ListImages
                  - ecr:CompleteLayerUpload
                  - ecr:BatchCheckLayerAvailability
                Resource: !Sub arn:aws:ecr:ap-northeast-1:${AccountId}:repository/${RepositoryName}
  GithubOidc:
    Type: AWS::IAM::OIDCProvider
    Properties:
      Url: https://token.actions.githubusercontent.com
      ClientIdList: 
        - sts.amazonaws.com
      ThumbprintList:
        - 6938fd4d98bab03faadb97b34396831e3780aea1
~~~

必要な権限については、多分これが最小構成だと思います。

- 認証
  - `ecr:GetAuthorizationToken`
- イメージの取得（キャッシュ利用時に使う）
  - `ecr:GetDownloadUrlForLayer`
  - `ecr:BatchGetImage`
- イメージの更新
  - `ecr:InitiateLayerUpload`
  - `ecr:PutImage`
  - `ecr:UploadLayerPart`
  - `ecr:ListImages`
  - `ecr:CompleteLayerUpload`
  - `ecr:BatchCheckLayerAvailability`

また、`ThumbprintList`は書き変わることが[あるらしいです](https://github.blog/changelog/2022-01-13-github-actions-update-on-oidc-based-deployments-to-aws/)。上記の公式テンプレートに倣えば基本は大丈夫そうですが、留意しておく必要はありそうです。

# ECRリポジトリの作成

こちらも CloudFormation で作ってしまいます。

~~~yml:cfn-ecr.yml
AWSTemplateFormatVersion: "2010-09-09"
Description: ECR resource
Parameters:
  ServiceName:
    Type: String
    Default: github-actions-ecr-push-test

Resources:
  ECR:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: !Ref ServiceName
      ImageScanningConfiguration:
        ScanOnPush: true
~~~

# Workflow の作成

コード全文はこのような感じです。

~~~yml:build.yml
name: code build
on: 
  workflow_dispatch:
  push:
    branches:
      - main
      - 'feature/**'
jobs:
  build:
    name: Build image
    env:
      SERVICE_NAME: github-actions-ecr-push-test
    runs-on: ubuntu-latest
    # These permissions are needed to interact with GitHub's OIDC Token endpoint.
    permissions:
      id-token: write
      contents: read
    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Cache Docker layers
      uses: actions/cache@v2
      with:
        path: /tmp/.buildx-cache
        key: ${{ runner.os }}-buildx-${{ github.sha }}
        restore-keys: |
          ${{ runner.os }}-buildx-

    - name: Configure AWS credentials from Test account
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
        aws-region: ap-northeast-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - uses: docker/build-push-action@v2
      id: build-image
      with:
        push: true
        file: deploy/development/Dockerfile
        tags: ${{ steps.login-ecr.outputs.registry }}/${{ env.SERVICE_NAME }}:latest
        cache-from: type=local,src=/tmp/.buildx-cache
        cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max

    - name: Move cache
      run: |
        rm -rf /tmp/.buildx-cache
        mv /tmp/.buildx-cache-new /tmp/.buildx-cache
~~~

詳しく見ていきます。

~~~yml
on: 
  workflow_dispatch:
  push:
    branches:
      - main
      - 'feature/**'
~~~

まず `on:` ブロックでは、GitHub Actions の起動条件を設定します。featureブランチにpushされたら自動で job が実行されるようなっており、feature ブランチに何かしらの更新をpushしたとき、自動で開発環境に反映させるような流れをイメージしています。

ここでmainブランチも指定しているのは、GitHub Action のキャッシュには「スコープ」という概念があり、デフォルトブランチでキャッシュを残しておく必要があるためです。

> 参考: https://zenn.dev/mallowlabs/articles/github-actions-cache-scope

`workflow_dispatch:` はUIから手動実行できるようにするオプションです。

以下の `jobs: build:` の処理については、[`docker/build-push-action@v2`](https://github.com/docker/build-push-action) という上手いことキャッシュを利用しながら build + push を行ってくれる GitHub Action に乗っかっている感じです。

キャッシュ周りは [local-cache](https://github.com/docker/build-push-action/blob/master/docs/advanced/cache.md#local-cache) パターンのテンプレートをECR用に書き換えています。

また、`AWS_ACCOUNT_ID` については Secrets から読み込むようにしているので、

~~~yml
    - name: Configure AWS credentials from Test account
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
        aws-region: ap-northeast-1
~~~

リポジトリの Settings 欄から設定しておく必要があります。

![](https://storage.googleapis.com/zenn-user-upload/b82d7a653d72-20220410.png)

# おわりに

キャッシュを利用することで、`bundle install` などで5分ほどかかっていたビルド時間を短縮することができました。

![](https://storage.googleapis.com/zenn-user-upload/10e407eb0f30-20220410.png)