---
title: "ECR"
---

この章では Docker イメージを置いておく、ECR リソースを作ります。

![](https://storage.googleapis.com/zenn-user-upload/9f916602c40d-20220430.png)

# ECR

このリソースは短いので最初からまとめて書きます。

`ImageScanningConfiguration:` は `ScanOnPush: true` と設定して、自動で脆弱性チェックをしてもらうようにします。これは無料で実行されます。

古いイメージを自動で削除していきたいので、`LifecyclePolicy:` も設定しておきます。ECR のリポジトリに保存されているイメージが10個以上になったら、古いほうから削除されます。

~~~yml:deploy/templates/cfn-ecr.yml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  ServiceName:
    Type: String
    Default: cfn-sample
  EnvSuffix:
    Type: String
    Default: -dev
    AllowedValues:
      - -dev
      - -qa
      - ''
Resources:
  ECR:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
      ImageScanningConfiguration:
        ScanOnPush: true
      LifecyclePolicy:
        LifecyclePolicyText: '{"rules":[{"rulePriority":1,"description":"Delete untagged more than 10 images.","selection":{"tagStatus":"untagged","countType":"imageCountMoreThan","countNumber":10},"action":{"type":"expire"}}]}'
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ecr-repository.html

# スタックの作成

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-ecr.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの作成

~~~
aws cloudformation create-stack --stack-name cfn-ecr --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-ecr.yml
~~~

# Push

ECR にイメージを push してみます。`{ACCOUNT_ID}` は各自の AWS アカウントID で読み替えてください。

~~~
# 認証
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin {ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com
~~~

`Dockerfile` が root に置かれていないので、`-f` オプションでファイルの場所を指定します。

~~~
docker build -f deploy/development/Dockerfile -t cfn-sample-dev .
docker tag cfn-sample-dev:latest {ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/cfn-sample-dev:latest
docker push {ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/cfn-sample-dev:latest
~~~

無事に反映されたら完了です。