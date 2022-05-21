---
title: "CloudWatch Logs / S3"
---

この章ではログを置く場所を作ります。リソースとしては S3 と CloudWatch Logs になります。

![](https://storage.googleapis.com/zenn-user-upload/31d30631d118-20220429.png)



# Parameters

ここからは、リソースにつける任意の名称を `ServiceName` として設定します。この記事では `cfn-sample` とします。`EnvSuffix:` は付ける必要はないのですが、CloudFormation を使うときはだいたい複数環境を作ることになるので、練習のために設定しておきます。 

~~~yml:deploy/templates/cfn-logs.yml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  ServiceName:
    Type: String
    Default: cfn-sample # 任意の名称を設定
  EnvSuffix:
    Type: String
    Default: -dev
    AllowedValues:
      - -dev # 開発環境
      - -qa  # 検証環境
      - ''   # 本番環境
~~~


# CloudWatch Logs

CloudWatch Logs にロググループを作ります。

- `ECSLogGroup`
  - まずは全ての ECS ログをここに吐くようにします。
- `FireLensLogGroup`
  - FireLens ログの出力先です。のちのち使います。

ログの保持期間は2週間にしています。

~~~yml:deploy/templates/cfn-logs.yml
  ECSLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub /aws/ecs/${ServiceName}${EnvSuffix}
      RetentionInDays: 14
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
  
  FireLensLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub /aws/ecs/${ServiceName}-firelens${EnvSuffix}
      RetentionInDays: 14
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-logs-loggroup.html

# S3

ログを保存する用のバケットを作ります。S3 のログに対しては、Athena からクエリを発行できたりします。

`VersioningConfiguration` はファイルのバージョン管理を有効にするかのオプションです。間違えて上書きしたり削除してしまった時とかに元に戻すことができます。今回は無効 `Suspended` にしていますが、有効にしたいときは `Enabled` と指定します。

`LifecycleConfiguration:` はライフサイクルポリシーを設定するオプションです。一定期間を過ぎたオブジェクトを自動で削除したいので、今回は有効期限 `ExpirationInDays:` を `30` 日として、1カ月経ったら削除されるようにします。ログは `fluent-bit-logs/tweet-log/` に保存する予定なので、その通りに `Prefix:` を設定しています。

~~~yml:deploy/templates/cfn-logs.yml
Resources:
  LogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Join [ '', [ !Ref ServiceName, -logs, !Ref EnvSuffix ] ]
      VersioningConfiguration:
        Status: Suspended
      LifecycleConfiguration:
        Rules:
          - Id: !Join [ '', [ !Ref ServiceName, -tweet-logs-lifecycle-policy, !Ref EnvSuffix ] ]
            Prefix: fluent-bit-logs/tweet-log/
            Status: Enabled
            ExpirationInDays: 30
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html

# 完成コード

:::details deploy/templates/cfn-logs.yml
~~~yml:deploy/templates/cfn-logs.yml
AWSTemplateFormatVersion: '2010-09-09'
Description: s3 resource
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
  ECSLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub /aws/ecs/${ServiceName}${EnvSuffix}
      RetentionInDays: 14
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
  
  FireLensLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub /aws/ecs/${ServiceName}-firelens${EnvSuffix}
      RetentionInDays: 14
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName

  LogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Join [ '', [ !Ref ServiceName, -logs, !Ref EnvSuffix ] ]
      VersioningConfiguration:
        Status: Suspended
      LifecycleConfiguration:
        Rules:
          - Id: !Join [ '', [ !Ref ServiceName, -tweet-logs-lifecycle-policy, !Ref EnvSuffix ] ]
            Prefix: fluent-bit-logs/tweet-log/
            Status: Enabled
            ExpirationInDays: 30
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~
:::

# スタックの作成

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-logs.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの作成

~~~
aws cloudformation create-stack --stack-name cfn-logs --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-logs.yml
~~~

ここではパラメータを指定していないので、パラメータは `Default` で指定したものになります。（`ServiceName: cfn-sample`, `EnvSuffix: -dev`）

別の値にしたいときは、`--parameters` で直接記述したり、

~~~
aws cloudformation create-stack --stack-name cfn-logs-qa --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-logs.yml --parameters ParameterKey=EnvSuffix,ParameterValue=-qa
~~~

パラメータが多くなったりしたときは json ファイルに書いてそれを指定したりします。

~~~
aws cloudformation create-stack --stack-name cfn-logs-qa --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-logs.yml --parameters file://deploy/templates/params/qa/cfn-logs-params.json
~~~

~~~json:deploy/templates/params/qa/cfn-logs-params.json
[
  {
    "ParameterKey": "EnvSuffix",
    "ParameterValue": "-qa"
  }
] 
~~~