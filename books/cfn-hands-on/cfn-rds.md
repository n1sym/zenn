---
title: "RDS / Secrets Manager / SSM Parameter Store "
---

この章では RDS リソースを作ります。

また、秘匿情報を置いておくための場所として、Secrets Manager を、パラメータを置いておく場所として Systems Manager Parameter Store を利用します。

![](https://storage.googleapis.com/zenn-user-upload/8c0ccc2a4939-20220430.png)

# 注意

:::message alert
DBインスタンスの維持には、24時間あたりで **1$ ほどの料金が発生します**。ご注意ください。
:::

# Parameters

RDS のスペックである `InstanceType` は最小の `db.t3.micro` を選びます。

~~~yml:deploy/templates/cfn-rds.yml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  ServiceName:
    Type: String
    Default: cfn-sample
  DatabaseName:
    Type: String
    Default: myapp
  EnvName:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - qa
      - prod
  EnvSuffix:
    Type: String
    Default: -dev
    AllowedValues:
      - -dev
      - -qa
      - ''
  DatabaseEnvSuffix:
    Type: String
    Default: _development
    AllowedValues:
      - _development
      - ''
  InstanceType:
    Type: String
    Default: db.t3.micro
~~~

# DBSubnetGroup

まずはDBサブネットグループを作成します。`SubnetIds:` は4章で作ったプライベートサブネットを参照しています。

~~~yml:deploy/templates/cfn-rds.yml
  DBSubnetGroup: 
    Type: AWS::RDS::DBSubnetGroup
    Properties: 
      DBSubnetGroupDescription: subnet group for db
      SubnetIds: 
        - !ImportValue cfn-base:PrivateSubnetDb1A
        - !ImportValue cfn-base:PrivateSubnetDb1C
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName 
~~~
https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbsubnet-group.html

# DBPassword

Secrets Manager で DB のパスワードを生成します。パスワードに対応していない記号などがあるので、`ExcludeCharacters:` で忘れずに設定しておきます。

> The password for the master user. The password can include any printable ASCII character except "/", """, or "@".
> 
> https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html#cfn-rds-dbinstance-masteruserpassword

~~~yml:deploy/templates/cfn-rds.yml
  DBPassword:
    Type: AWS::SecretsManager::Secret
    Properties:
      GenerateSecretString:
        PasswordLength: 16
        ExcludeCharacters: '"@/\'
      Name: !Sub ${ServiceName}${EnvSuffix}-database-password
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-secretsmanager-secret.html

# DBInstance

DBのインスタンスについての設定項目はかなりたくさんあるのですが、今回は必要最低限の項目のみ設定していきます。

まずは [DeletionPolicy](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html) 属性についてです。これはスタックが削除された際にリソースを保持するか、場合によってはバックアップを残すかを設定する項目です。

`AWS::RDS::DBInstance` の `DeletionPolicy:` はデフォルトで `Snapshot` となっており、スタックが削除された際に自動でバックアップが残るようになっています。普段使いの時はこのままでもよいのですが、バックアップストレージにも [料金は発生する](https://aws.amazon.com/jp/rds/postgresql/pricing/?pg=pr&loc=3) ので、今回は `Delete` としています。なので、スタックを削除した際にバックアップが残ることはありません。

| Properties | 説明 |
| ---- | ---- |
| `Engine:` | `postgres` を採用しています。|
| `MasterUsername:` | データベースのユーザー名を指定します。2章の Rails セットアップで設定した名前と合わせて、ユーザー名は `postgres` としておきます。 |
| `MasterUserPassword:` | 先ほど作った `DBPassword` を参照しています。参照の仕方にややクセがあるので、[ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/dynamic-references.html) を参考にしつつ設定します。|
| `DBName:` | データベース名。Rails のデフォルト設定に合わせて `myapp_development` としています。|
| `DBInstanceIdentifier:` | インスタンス名。分かりやすい名前を設定しておきます。 |
| `DBInstanceClass:` | インスタンスタイプ。Parameters で最小スペックである `db.t3.micro` を指定しています。|
| `AllocatedStorage:` | DB インスタンスに割り当てる初期ストレージの大きさ。最小値である `20` を設定しています。 |
| `BackupRetentionPeriod:` | 自動バックアップを保持する日数。デフォルトは `1` と設定されていますが、今回はバックアップストレージの料金を発生させたくないので、`0` と設定しておき、自動バックアップを無効にしています。 |


~~~yml:deploy/templates/cfn-rds.yml
  DBInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Delete
    Properties:
      Engine: postgres
      MasterUsername: postgres
      MasterUserPassword: !Sub '{{resolve:secretsmanager:${ServiceName}${EnvSuffix}-database-password:SecretString}}'
      DBName: !Join [ '', [ !Ref DatabaseName, !Ref DatabaseEnvSuffix ] ]
      DBInstanceIdentifier:  !Sub ${ServiceName}${EnvSuffix}-instance-1
      DBInstanceClass: !Ref InstanceType
      AllocatedStorage: 20
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups: !ImportValue cfn-base:SecurityGroupDbId
      BackupRetentionPeriod: 0
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html

# DBInstanceEndpoint

ECS のタスク定義に設定する環境変数から、データベースのエンドポイントを参照したいので SSM Parameter Store に値を保存しておきます。

~~~yml:deploy/templates/cfn-rds.yml
  DBInstanceEndpoint:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub /${ServiceName}/${EnvName}/DB_HOST
      Type: String
      Value: !GetAtt DBInstance.Endpoint.Address
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ssm-parameter.html

# Parameter Store vs Secrets Manager

どちらも秘匿情報を保存しておくことができる場所ですが、どちらを使うのがよいでしょうか。比較記事としては以下が詳しいと思います。

https://qiita.com/tomoya_oka/items/a3dd44879eea0d1e3ef5

公式の回答としては、以下がありました。

> Q: Parameter Store と Secrets Manager のどちらを使えば良いですか?
> 
> 設定とシークレットにひとつのストアが欲しい場合、パラメータストアをお使いください。ライフサイクル管理を備えたシークレット専用のストアが欲しい場合、シークレットマネージャーをお使いください。パラメータストアは追加料金なしで、パラメータ数 10,000 個の制限でお使いいただけます。詳細については、Secrets Manager の料金ページを参照してください。

https://aws.amazon.com/jp/systems-manager/faq/#Parameter_Store

ただ、パラメータストアの SecureString 形式については、現在の CloudFormation では[対応していない](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ssm-parameter.html#cfn-ssm-parameter-type) ということがあり、秘匿情報を CloudFormation で作成するなら Secrets Manager を使うことになると思います。

Secrets Manager はシークレット1件から有料だったりするので、単に何らかのパラメータを参照できるようにしておきたい場合は、パラメータストアを使うのがよいと思います。

# Outputs

データベースのパスワードを置いている場所は、ECS のタスク定義と IAM ロールから参照したいので Outputs で出力しておきます。

~~~yml:deploy/templates/cfn-rds.yml
Outputs:
  SecretsManagerDBPasswordArn:
    Value: !Ref DBPassword
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, SecretsManagerDBPasswordArn ] ]
~~~

# 完成コード

:::details deploy/templates/cfn-rds.yml
~~~yml:deploy/templates/cfn-rds.yml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  ServiceName:
    Type: String
    Default: cfn-sample
  DatabaseName:
    Type: String
    Default: myapp
  EnvName:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - qa
      - prod
  EnvSuffix:
    Type: String
    Default: -dev
    AllowedValues:
      - -dev
      - -qa
      - ''
  DatabaseEnvSuffix:
    Type: String
    Default: _development
    AllowedValues:
      - _development
      - ''
  InstanceType:
    Type: String
    Default: db.t3.micro

Resources:
  DBSubnetGroup: 
    Type: AWS::RDS::DBSubnetGroup
    Properties: 
      DBSubnetGroupDescription: subnet group for db
      SubnetIds: 
        - !ImportValue cfn-base:PrivateSubnetDb1A
        - !ImportValue cfn-base:PrivateSubnetDb1C
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName

  DBPassword:
    Type: AWS::SecretsManager::Secret
    Properties:
      GenerateSecretString:
        PasswordLength: 16
        ExcludeCharacters: '"@/\'
      Name: !Sub ${ServiceName}${EnvSuffix}-database-password

  DBInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Delete
    Properties:
      Engine: postgres
      MasterUsername: postgres
      MasterUserPassword: !Sub '{{resolve:secretsmanager:${ServiceName}${EnvSuffix}-database-password:SecretString}}'
      DBName: !Join [ '', [ !Ref DatabaseName, !Ref DatabaseEnvSuffix ] ]
      DBInstanceIdentifier:  !Sub ${ServiceName}${EnvSuffix}-instance-1
      DBInstanceClass: !Ref InstanceType
      AllocatedStorage: 20
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups: 
        - !ImportValue cfn-base:SecurityGroupDbId
      BackupRetentionPeriod: 0
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName

  DBInstanceEndpoint:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub /${ServiceName}/${EnvName}/DB_HOST
      Type: String
      Value: !GetAtt DBInstance.Endpoint.Address

Outputs:
  SecretsManagerDBPasswordArn:
    Value: !Ref DBPassword
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, SecretsManagerDBPasswordArn ] ]
~~~
:::

# スタックの作成

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-rds.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの作成

~~~
aws cloudformation create-stack --stack-name cfn-rds --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-rds.yml
~~~

無事に作成されたら完了です。



:::message 
一旦中断する場合は、忘れずにリソースの削除をしておきましょう。
:::