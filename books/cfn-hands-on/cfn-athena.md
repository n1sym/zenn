---
title: "Athena"
---

AWS Athena を利用して、前の章でアップロードしたログに対してクエリを発行してみます。

![](https://storage.googleapis.com/zenn-user-upload/889b6318f25a-20220516.png)

# Athena リソースの作成

Partition Projection を使用して、パーティション管理を自動化しています。

https://dev.classmethod.jp/articles/20200627-amazon-athena-partition-projection/

~~~yml:deploy/templates/cfn-athena.yml
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
  GlueDatabase:
    Type: AWS::Glue::Database
    Properties:
      CatalogId: !Ref AWS::AccountId
      DatabaseInput:
        Name: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix, -glue-database ] ]

  GlueTable:
    Type: AWS::Glue::Table
    Properties:
      CatalogId: !Ref AWS::AccountId
      DatabaseName: !Ref GlueDatabase
      TableInput:
        Name: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix, -glue-table ] ]
        TableType: EXTERNAL_TABLE
        Parameters:
          has_encrypted_data: false
          serialization.encoding: utf-8
          EXTERNAL: true
          projection.enabled: true
          projection.dt.type: date
          projection.dt.range: "2022/04/01,NOW"
          projection.dt.format: yyyy/MM/dd
          projection.dt.interval: 1
          projection.dt.interval.unit: DAYS
          storage.location.template:
            !Join
              - ''
              - - !Sub s3://${ServiceName}-logs${EnvSuffix}/fluent-bit-logs/tweet-log/
                - '${dt}'
        StorageDescriptor:
          OutputFormat: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
          Columns:
            - Name: date
              Type: date
            - Name: ip
              Type: string
            - Name: environment
              Type: string
            - Name: level
              Type: string
            - Name: title
              Type: string
            - Name: content
              Type: string
            - Name: user
              Type: string
            - Name: message
              Type: string
          InputFormat: org.apache.hadoop.mapred.TextInputFormat
          Location: !Sub s3://${ServiceName}-logs${EnvSuffix}/fluent-bit-logs/tweet-log
          SerdeInfo:
            Parameters:
              serialization.format: '1'
            SerializationLibrary: org.openx.data.jsonserde.JsonSerDe
        PartitionKeys:
          - Name: dt
            Type: string
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-glue-database.html

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-glue-table.html

https://docs.aws.amazon.com/ja_jp/athena/latest/ug/partition-projection-supported-types.html


# スタックの作成

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-athena.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの作成

~~~
aws cloudformation create-stack --stack-name cfn-athena --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-athena.yml
~~~

# AWS Athena の初期設定

はじめて AWS Athena を使う場合は、クエリ結果を保存する s3 バケットを設定する必要があります。

![](https://storage.googleapis.com/zenn-user-upload/da7564b301e4-20220518.png)

設定タブから管理ページへ移動して、

![](https://storage.googleapis.com/zenn-user-upload/716e2cb56a66-20220518.png)

任意の s3 バケットのパスを指定します。

![](https://storage.googleapis.com/zenn-user-upload/b9f3ac7b7d05-20220518.png)


これで準備は完了です。

# 動作確認

Athena で実際にクエリを実行してみます。

~~~
SELECT * FROM "cfn-sample-dev-glue-database"."cfn-sample-dev-glue-table" limit 10;
~~~

結果が表示されたら成功です。

![](https://storage.googleapis.com/zenn-user-upload/da61a2e90845-20220518.png)


パーティションが効いているので、`where dt =` を指定してクエリを発行するとスキャンするデータ範囲が減ったりします。（複数日のログがあることが前提ですが）

~~~
SELECT * FROM "cfn-sample-dev-glue-database"."cfn-sample-dev-glue-table" where dt = '2022/05/16';
~~~

:::message 
一旦中断する場合は、忘れずにリソースの削除をしておきましょう。
:::