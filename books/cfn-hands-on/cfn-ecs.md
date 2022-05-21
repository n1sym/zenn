---
title: "ECS / ALB"
---
この章では ECS と ALB を作ります。これで実際に AWS にデプロイしたアプリに接続できるようになります。また、ECS Exec でコンテナに入ってコマンドが打てることを確認します。

![](https://storage.googleapis.com/zenn-user-upload/211ef69e31bd-20220503.png)

# 注意

:::message alert
ALB の維持には、24時間あたりで **0.5$ ほどの料金が発生します**。
ECS の維持には、24時間あたりで **1.5$ ほどの料金が発生します**。
:::

# Parameters

~~~yml:deploy/templates/cfn-ecs.yml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  ServiceName:
    Type: String
    Default: cfn-sample
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
~~~

# ALB

アプリケーションロードバランサーを作ります。

今回は ECS をパブリックサブネットに置くので、`Scheme:` は `internet-facing` を指定します。

~~~yml:deploy/templates/cfn-ecs.yml
  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      IpAddressType: ipv4
      Name: !Join [ '', [ alb-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Scheme: internet-facing
      SecurityGroups:
        - !ImportValue cfn-base:SecurityGroupIngressId
      Subnets:
        - !ImportValue cfn-base:PublicSubnetIngress1A
        - !ImportValue cfn-base:PublicSubnetIngress1C
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-loadbalancer.html

# TargetGroup

Blue/Green デプロイを行いたいので、ターゲットグループは Blue と Green の2つ作ります。 

ヘルスチェックの間隔 `HealthCheckIntervalSeconds:` は、60秒、タイムアウトまでの期限 ` HealthCheckTimeoutSeconds:` は5秒に設定しています。

`HealthyThresholdCount:` を 2 に設定しているので、ヘルスチェックが2回成功したら、タスクが正常に起動したものと判定されます。

~~~yml:deploy/templates/cfn-ecs.yml
  TargetGroupGreen:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckIntervalSeconds: 60
      HealthCheckPath: /api/health_check
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 2
      TargetType: ip
      Name: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix, -tg-green ] ]
      Port: 80
      Protocol: HTTP
      VpcId: !ImportValue cfn-base:VpcId
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
  TargetGroupBlue:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckIntervalSeconds: 60
      HealthCheckPath: /api/health_check
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 2
      TargetType: ip
      Name: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix, -tg-blue ] ]
      Port: 80
      Protocol: HTTP
      VpcId: !ImportValue cfn-base:VpcId
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-targetgroup.html

# ALBListener

アプリケーションロードバランサーに紐づける、本稼働リスナー `Port: 80` と、テストリスナー `Port: 10080` の2つを作ります。

~~~yml:deploy/templates/cfn-ecs.yml
  ALBListener1:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - TargetGroupArn: !Ref TargetGroupGreen
          Type: forward
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP

  ALBListener2:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - TargetGroupArn: !Ref TargetGroupGreen
          Type: forward
      LoadBalancerArn: !Ref ALB
      Port: 10080
      Protocol: HTTP
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-listener.html

# ECSCluster

ECS クラスターを作ります。

ECS 関連の詳しいメトリクスを取得できる `containerInsights` を有効 `enabled` にしています。ただ、この機能は**追加課金の対象となる**ので、必要ない場合は無効にしてもよいと思います。

~~~yml:deploy/templates/cfn-ecs.yml
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
      ClusterSettings:
        - Name: containerInsights
          Value: enabled
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-cluster.html

# ECSService

ECS サービスを作ります。`DependsOn` で前提となるリソースを指定しています。 

| Properties | 説明 |
| ---- | ---- |
| `Cluster:` | ↑で作ったクラスターを指定 |
| `ServiceName:` | サービス名を入力 |
| `TaskDefinition:` | タスク定義を指定 |
| `DeploymentController:` | デプロイ方法を指定するところ。今回は B/G デプロイをしたいので`CODE_DEPLOY`を指定しています。 |
| `DesiredCount:` | Rails サーバーを1つ起動すればいいので、`1`とします。 |
| `EnableExecuteCommand:` | ECS Exec を有効にしたいので、`true`とします。 |
| `HealthCheckGracePeriodSeconds:` | ヘルスチェックの猶予期間を設定します。60秒間隔のヘルスチェックで2回成功する必要があるので、ちょっと余裕を持たせて `300` 秒としています。 |
| `LaunchType:` | タスクの起動タイプを指定するところ。`FARGATE` を指定します。 |
| `AssignPublicIp:` | 今回は簡易のため、パブリックサブネットにアプリケーションを公開します。デフォルトでは無効 `DISABLED` になっていますが、パブリックIPの割り当てを有効 `ENABLED` にします。パブリックIPを割り当てないパターンは13章で扱っています。|

~~~yml:deploy/templates/cfn-ecs.yml
  ECSService:
    Type: AWS::ECS::Service
    DependsOn:
      - ALBListener1
      - ALBListener2
    Properties:
      Cluster: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
      ServiceName: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
      TaskDefinition: !Sub '{{resolve:ssm:/${ServiceName}/${EnvName}/ECSTaskDefinitionArn}}'
      DeploymentController:
        Type: CODE_DEPLOY
      DesiredCount: 1
      EnableExecuteCommand: true
      HealthCheckGracePeriodSeconds: 300
      LaunchType: FARGATE
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: ENABLED
          SecurityGroups:
            - !ImportValue cfn-base:SecurityGroupEcsId
          Subnets:
            - !ImportValue cfn-base:PublicSubnetIngress1A
            - !ImportValue cfn-base:PublicSubnetIngress1C
      LoadBalancers:
        - ContainerName: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
          ContainerPort: 80
          TargetGroupArn: !Ref TargetGroupGreen
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html

# Outputs

バッチのリソースを作る際に必要なので、ECS クラスターの Arn を出力しておきます。

~~~yml:deploy/templates/cfn-ecs.yml
  ECSClusterArn:
    Value: !GetAtt ECSCluster.Arn
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, ECSClusterArn ] ]
~~~

# 完成コード

:::details deploy/templates/cfn-ecs.yml
~~~yml:deploy/templates/cfn-ecs.yml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  ServiceName:
    Type: String
    Default: cfn-sample
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

Resources:
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
      ClusterSettings:
        - Name: containerInsights
          Value: enabled
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName

  ECSService:
    Type: AWS::ECS::Service
    DependsOn:
      - ALBListener1
      - ALBListener2
    Properties:
      Cluster: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
      DeploymentController:
        Type: CODE_DEPLOY
      DesiredCount: 1
      EnableExecuteCommand: true
      HealthCheckGracePeriodSeconds: 300
      LaunchType: FARGATE
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: ENABLED
          SecurityGroups:
            - !ImportValue cfn-base:SecurityGroupEcsId
          Subnets:
            - !ImportValue cfn-base:PublicSubnetIngress1A
            - !ImportValue cfn-base:PublicSubnetIngress1C
      LoadBalancers:
        - ContainerName: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
          ContainerPort: 80
          TargetGroupArn: !Ref TargetGroupGreen
      TaskDefinition: !Sub '{{resolve:ssm:/${ServiceName}/${EnvName}/ECSTaskDefinitionArn}}'
      ServiceName: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
  
  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      IpAddressType: ipv4
      Name: !Join [ '', [ alb-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Scheme: internet-facing
      SecurityGroups:
        - !ImportValue cfn-base:SecurityGroupIngressId
      Subnets:
        - !ImportValue cfn-base:PublicSubnetIngress1A
        - !ImportValue cfn-base:PublicSubnetIngress1C
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName

  TargetGroupGreen:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckIntervalSeconds: 60
      HealthCheckPath: /api/health_check
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 2
      TargetType: ip
      Name: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix, -tg-green ] ]
      Port: 80
      Protocol: HTTP
      VpcId: !ImportValue cfn-base:VpcId
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
  TargetGroupBlue:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckIntervalSeconds: 60
      HealthCheckPath: /api/health_check
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 2
      TargetType: ip
      Name: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix, -tg-blue ] ]
      Port: 80
      Protocol: HTTP
      VpcId: !ImportValue cfn-base:VpcId
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
  
  ALBListener1:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - TargetGroupArn: !Ref TargetGroupGreen
          Type: forward
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP

  ALBListener2:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - TargetGroupArn: !Ref TargetGroupGreen
          Type: forward
      LoadBalancerArn: !Ref ALB
      Port: 10080
      Protocol: HTTP

Outputs:
  ECSClusterArn:
    Value: !GetAtt ECSCluster.Arn
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, ECSClusterArn ] ]
~~~
:::

# スタックの作成

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-ecs.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの作成

~~~
aws cloudformation create-stack --stack-name cfn-ecs --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-ecs.yml
~~~

# 動作確認

## 接続テスト

アプリケーションが公開されているアドレス（`DNSName`）を取得します。EC2 > ロードバランサー から ALB の詳細画面を開いても取得できます。

~~~
aws elbv2 describe-load-balancers --query "LoadBalancers[].{ALBName: LoadBalancerName, DNSName: DNSName}"
~~~

Rails サーバーの立ち上げに 1~2 分かかるので、少し待ってから接続するとアプリケーションにアクセスして操作できるようになると思います。

## ECS Exec

ECS Exec でコンテナに入ってコマンドが打てることを確認します。AWS CLI 用の Session Manager プラグインがインストールされていることを前提としています。

https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html



タスクID（以下のレスポンスの `df9136f8f7b74388b523bc103afcd79d` の部分）を取得して、
~~~
aws ecs list-tasks --cluster cfn-sample-dev
{
    "taskArns": [
        "arn:aws:ecs:ap-northeast-1:668672929603:task/cfn-sample-dev/df9136f8f7b74388b523bc103afcd79d"
    ]
}
~~~

ECS Exec を実行します。`{TASK_ID}` は 各々のタスク ID に読み替えてください。
~~~
aws ecs execute-command --cluster cfn-sample-dev --task {TASK_ID} --container cfn-sample-dev --interactive --command '/bin/bash'
~~~

接続に成功したら任意の bash コマンドを入力できると思います。`bundle exec rails c` でコンソールに入ることも可能です。

## ログの確認

ログは、CloudWatch > Log groups の `/aws/ecs/cfn-sample-dev` に出力されています。ECS のタスク詳細画面にもリンクがあるので、そこから飛ぶことも可能です。

:::message 
一旦中断する場合は、忘れずにリソースの削除をしておきましょう。
:::

# おまけ：まとめてスタックを削除するシェルスクリプト

放置するだけで料金が発生するリソース類をまとめて削除するやつです。

ECS関連リソース は1回目の削除が失敗することがある（タスクが実行中だったり、ターゲットグループの向きが変わってCFnスタック管理対象外になったり）ので最後にもう一度 `delete-stack` を送っています。

~~~sh:deploy/templates/cleanup.sh
echo 'cleanup start'

echo 'task stack delete...'
aws cloudformation delete-stack --stack-name cfn-task
aws cloudformation wait stack-delete-complete --stack-name cfn-task
echo 'task stack deleted !'

echo 'iam stack delete...'
aws cloudformation delete-stack --stack-name cfn-iam
aws cloudformation wait stack-delete-complete --stack-name cfn-iam
echo 'iam stack deleted !'

echo 'rds stack delete...'
aws cloudformation delete-stack --stack-name cfn-rds

echo 'ecs stack delete...'
aws cloudformation delete-stack --stack-name cfn-ecs

aws cloudformation wait stack-delete-complete --stack-name cfn-rds
aws cloudformation wait stack-delete-complete --stack-name cfn-ecs

echo 'Re: ecs stack delete...'
aws cloudformation delete-stack --stack-name cfn-ecs

echo 'cleanup finished'
~~~

もう一度リソースを作りたいときは以下を実行します。

~~~sh:deploy/templates/setup.sh
echo 'setup start'

echo 'rds stack create...'
aws cloudformation create-stack --stack-name cfn-rds --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-rds.yml
aws cloudformation wait stack-create-complete --stack-name cfn-rds
echo 'rds stack created !'

echo 'iam stack create...'
aws cloudformation create-stack --stack-name cfn-iam --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-iam.yml --capabilities CAPABILITY_NAMED_IAM
aws cloudformation wait stack-create-complete --stack-name cfn-iam
echo 'iam stack created !'

echo 'task stack create...'
aws cloudformation create-stack --stack-name cfn-task --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-task.yml
aws cloudformation wait stack-create-complete --stack-name cfn-task
echo 'task stack created !'

echo 'ecs stack create...'
aws cloudformation create-stack --stack-name cfn-ecs --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-ecs.yml
aws cloudformation wait stack-create-complete --stack-name cfn-ecs
echo 'ecs stack created !'

echo 'setup finished'
~~~