---
title: "TaskDefinition"
---

この章では ECS のタスク定義を作ります。

![](https://storage.googleapis.com/zenn-user-upload/da73364f8c2c-20220501.png)

# IAMリソース

ECS 関連の IAM ロールを作っておきます。それぞれのロールに以下の権限を渡しています。

### `ECSTaskExecutionRole` (ECSに委譲するロール)
  - `ECSTaskExecutionPolicy`
    - ECR からイメージを取得
    - ログの出力
      - `Resource:` を特定のロググループに限定
  - `SecretReadOnlyPolicy`
    - SSM Parameter Store から値を取得
    - Secrets Manager から値を取得
      - `Resource:` を特定の Arn に限定

### `ECSTaskRole` (タスクに委譲するロール)
  - `ECSExecPolicy`
    - ECS Exec を行うための権限 (ssmmessages)

### `ECSCodeDeployRole` (CodeDeployに委譲するロール)
  - `AWSCodeDeployRoleForECS`
    - AWS側で管理されているポリシーをそのままアタッチしています。

### `ECSEventsRole` (イベントトリガーに委譲するロール）
- `ECSEventsPolicy`
  - タスクの実行

~~~yml:deploy/templates/cfn-iam.yml
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
  ECSTaskExecutionRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ecs-tasks.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      RoleName: !Join [ '', [ ecs-task-execution-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
      Policies:
        - PolicyName: ECSTaskExecutionPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'ecr:GetAuthorizationToken'
                  - 'ecr:BatchCheckLayerAvailability'
                  - 'ecr:GetDownloadUrlForLayer'
                  - 'ecr:BatchGetImage'
                Resource:
                  - '*'
              - Effect: Allow
                Action:
                  - 'logs:CreateLogGroup'
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                Resource:
                  - !Sub arn:aws:logs:ap-northeast-1:${AWS::AccountId}:log-group:/aws/ecs/${ServiceName}${EnvSuffix}:*
                  - !Sub arn:aws:logs:ap-northeast-1:${AWS::AccountId}:log-group:/aws/ecs/containerinsights/${ServiceName}${EnvSuffix}/performance:*
        - PolicyName: SecretReadOnlyPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'ssm:Describe*'
                  - 'ssm:Get*'
                  - 'ssm:List*'
                Resource: '*'
              - Effect: Allow
                Action:
                  - 'secretsmanager:GetResourcePolicy*'
                  - 'secretsmanager:GetSecretValue'
                  - 'secretsmanager:DescribeSecret'
                  - 'secretsmanager:ListSecretVersionIds'
                Resource: !ImportValue cfn-rds:SecretsManagerDBPasswordArn
  ECSTaskRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ecs-tasks.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      RoleName: !Join [ '', [ ecs-task-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
      Policies:
        - PolicyName: ECSExecPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'ssmmessages:CreateControlChannel'
                  - 'ssmmessages:CreateDataChannel'
                  - 'ssmmessages:OpenControlChannel'
                  - 'ssmmessages:OpenDataChannel'
                Resource:
                  - '*'
  
  ECSCodeDeployRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - codedeploy.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      RoleName: !Join [ '', [ ecs-code-deploy-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS
  
  ECSEventsRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - events.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      RoleName: !Join [ '', [ ecs-events-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
      Policies:
        - PolicyName: ECSEventsPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'ecs:RunTask'
                Resource: '*'
              - Effect: Allow
                Action:
                  - iam:PassRole
                Resource: '*'
                Condition:
                  StringLike:
                    iam:PassedToService: ecs-tasks.amazonaws.com

Outputs:
  ECSTaskExecutionRole:
    Value: !GetAtt ECSTaskExecutionRole.Arn
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, ECSTaskExecutionRole ] ]
  ECSTaskRole:
    Value: !GetAtt ECSTaskRole.Arn
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, ECSTaskRole ] ]
  ECSEventsRole:
    Value: !GetAtt ECSEventsRole.Arn
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, ECSEventsRole ] ]
~~~

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html

# タスク定義

ECS で Docker コンテナを実行するには、タスク定義が必要になります。

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task_definitions.html

## Parameters

~~~yml:deploy/templates/cfn-task.yml
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
  RailsEnv:
    Type: String
    Default: development
    AllowedValues:
      - development
      - production
~~~


## TaskDefinition

項目が多いですが、１つ１つ設定していきます。

| Properties | 説明 |
| ---- | ---- |
| `ContainerDefinitions:` | コンテナの定義を設定するフィールド |
| `Name:` | コンテナ名を指定 |
| `Image:` | 参照するイメージを指定。今回は5章で作成したリポジトリの最新イメージを取得しています。 | 
| `LogConfiguration:` | ログ出力の設定をするフィールド。今回はほとんどデフォルト設定のままです。 |
| `PortMappings:` | コンテナが送受信できるポート番号を指定します。|
| `Command:` | コンテナが起動した時に実行するコマンドを指定します。 |
| `Environment:` | 環境変数を設定するフィールド |
| `Secrets:` | 環境変数（秘匿情報）を設定するフィールド |
| `Cpu:` | コンテナに割り当てる CPU のスペックを指定。今回は最小値である `256` としています。 |
| `Memory:` | コンテナに割り当てるメモリのスペックを指定。今回は最小値である `512` としています。 |
| `ExecutionRoleArn:` | ECS に AWS の APIコール（イメージの取得や、ログの出力など）を代行してもらうので、ロールを委譲します。|
| `Family:` | タスク定義の名前を指定。 |
| `NetworkMode:` | Fargate でコンテナを起動したいので、`awsvpc` を指定する必要があります。 |
| `RequiresCompatibilities:` | タスクの起動タイプを指定するところ。`FARGATE` を指定します。 |
| `TaskRoleArn:` | タスクそのもの（実行しているコード）に委譲するロールです。ECS Exec をするために必要な権限を渡しています。後々 Rails アプリケーションから S3 にログを出力したりする時には s3 へのアップロード権限なども渡します。|

~~~yml:deploy/templates/cfn-task.yml
Resources:
  ECSTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      ContainerDefinitions:
        - Name: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
          Image: !Sub ${AWS::AccountId}.dkr.ecr.ap-northeast-1.amazonaws.com/${ServiceName}${EnvSuffix}:latest
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Sub /aws/ecs/${ServiceName}${EnvSuffix}
              awslogs-region: ap-northeast-1
              awslogs-stream-prefix: ecs
          PortMappings:
            - ContainerPort: 80
              HostPort: 80
              Protocol: tcp
          Command:
            - /app/entrypoint.sh
          Environment:
            - Name: DATABASE_USERNAME
              Value: postgres
            - Name: RAILS_LOG_TO_STDOUT
              Value: true
            - Name: RAILS_ENV
              Value: !Ref RailsEnv
            - Name: TZ
              Value: Asia/Tokyo
          Secrets:
            - Name: DATABASE_HOST
              ValueFrom: !Sub /${ServiceName}/${EnvName}/DB_HOST
            - Name: DATABASE_PASSWORD
              ValueFrom: !ImportValue cfn-rds:SecretsManagerDBPasswordArn
      Cpu: 256
      Memory: 512
      ExecutionRoleArn: !ImportValue cfn-iam:ECSTaskExecutionRole
      Family: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      TaskRoleArn: !ImportValue cfn-iam:ECSTaskExecutionRole
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html#cfn-ecs-taskdefinition-family

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ecs-taskdefinition-containerdefinitions.html

# ECSTaskDefinitionArn

ECS リソースから参照するため、タスク定義のリソースネーム (Arn) を SSM Parameter Store に保存しておきます。

~~~yml:deploy/templates/cfn-task.yml
  ECSTaskDefinitionArn:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub /${ServiceName}/${EnvName}/ECSTaskDefinitionArn
      Type: String
      Value: !Ref ECSTaskDefinition
~~~

# 完成コード

:::details deploy/templates/cfn-iam.yml
~~~yml:deploy/templates/cfn-iam.yml
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
  ECSTaskExecutionRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ecs-tasks.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      RoleName: !Join [ '', [ ecs-task-execution-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
      Policies:
        - PolicyName: ECSTaskExecutionPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'ecr:GetAuthorizationToken'
                  - 'ecr:BatchCheckLayerAvailability'
                  - 'ecr:GetDownloadUrlForLayer'
                  - 'ecr:BatchGetImage'
                Resource:
                  - '*'
              - Effect: Allow
                Action:
                  - 'logs:CreateLogGroup'
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                Resource:
                  - !Sub arn:aws:logs:ap-northeast-1:${AWS::AccountId}:log-group:/aws/ecs/${ServiceName}${EnvSuffix}:*
                  - !Sub arn:aws:logs:ap-northeast-1:${AWS::AccountId}:log-group:/aws/ecs/containerinsights/${ServiceName}${EnvSuffix}/performance:*
        - PolicyName: SecretReadOnlyPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'ssm:Describe*'
                  - 'ssm:Get*'
                  - 'ssm:List*'
                Resource: '*'
              - Effect: Allow
                Action:
                  - 'secretsmanager:GetResourcePolicy*'
                  - 'secretsmanager:GetSecretValue'
                  - 'secretsmanager:DescribeSecret'
                  - 'secretsmanager:ListSecretVersionIds'
                Resource: !ImportValue cfn-rds:SecretsManagerDBPasswordArn
  ECSTaskRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ecs-tasks.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      RoleName: !Join [ '', [ ecs-task-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
      Policies:
        - PolicyName: ECSExecPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'ssmmessages:CreateControlChannel'
                  - 'ssmmessages:CreateDataChannel'
                  - 'ssmmessages:OpenControlChannel'
                  - 'ssmmessages:OpenDataChannel'
                Resource:
                  - '*'
  
  ECSCodeDeployRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - codedeploy.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      RoleName: !Join [ '', [ ecs-code-deploy-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS
  
  ECSEventsRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - events.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      RoleName: !Join [ '', [ ecs-events-, !Ref ServiceName, !Ref EnvSuffix ] ]
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
      Policies:
        - PolicyName: ECSEventsPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'ecs:RunTask'
                Resource: '*'
              - Effect: Allow
                Action:
                  - iam:PassRole
                Resource: '*'
                Condition:
                  StringLike:
                    iam:PassedToService: ecs-tasks.amazonaws.com

Outputs:
  ECSTaskExecutionRole:
    Value: !GetAtt ECSTaskExecutionRole.Arn
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, ECSTaskExecutionRole ] ]
  ECSTaskRole:
    Value: !GetAtt ECSTaskRole.Arn
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, ECSTaskRole ] ]
  ECSEventsRole:
    Value: !GetAtt ECSEventsRole.Arn
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, ECSEventsRole ] ]
~~~
:::

:::details deploy/templates/cfn-task.yml
~~~yml:deploy/templates/cfn-task.yml
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
  RailsEnv:
    Type: String
    Default: development
    AllowedValues:
      - development
      - production

Resources:
  ECSTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      ContainerDefinitions:
        - Name: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
          Image: !Sub ${AWS::AccountId}.dkr.ecr.ap-northeast-1.amazonaws.com/${ServiceName}${EnvSuffix}:latest
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Sub /aws/ecs/${ServiceName}${EnvSuffix}
              awslogs-region: ap-northeast-1
              awslogs-stream-prefix: ecs
          PortMappings:
            - ContainerPort: 80
              HostPort: 80
              Protocol: tcp
          Command:
            - /app/entrypoint.sh
          Environment:
            - Name: DATABASE_USERNAME
              Value: postgres
            - Name: RAILS_LOG_TO_STDOUT
              Value: true
            - Name: RAILS_ENV
              Value: !Ref RailsEnv
            - Name: TZ
              Value: Asia/Tokyo
          Secrets:
            - Name: DATABASE_HOST
              ValueFrom: !Sub /${ServiceName}/${EnvName}/DB_HOST
            - Name: DATABASE_PASSWORD
              ValueFrom: !ImportValue cfn-rds:SecretsManagerDBPasswordArn
      Cpu: 256
      ExecutionRoleArn: !ImportValue cfn-iam:ECSTaskExecutionRole
      Family: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
      Memory: 512
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      TaskRoleArn: !ImportValue cfn-iam:ECSTaskRole
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
  
  ECSTaskDefinitionArn:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub /${ServiceName}/${EnvName}/ECSTaskDefinitionArn
      Type: String
      Value: !Ref ECSTaskDefinition
~~~
:::

# スタックの作成

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-iam.yml s3://cfn-sample-templates/services/cfn-sample/
aws s3 cp deploy/templates/cfn-task.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの作成

`cfn-task` スタックは `cfn-iam` スタックに依存しているので、作成されるまで少し間を開けます。

~~~
aws cloudformation create-stack --stack-name cfn-iam --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-iam.yml --capabilities CAPABILITY_NAMED_IAM
aws cloudformation create-stack --stack-name cfn-task --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-task.yml
~~~

無事に作成されたら完了です。

:::message 
一旦中断する場合は、忘れずにリソースの削除をしておきましょう。
:::