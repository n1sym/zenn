---
title: "Fluent Bit のセットアップ"
---

今回はログ周りの設定を行います。FireLens（Fluent Bit）を利用して、Rails のログを管理します。

![](https://storage.googleapis.com/zenn-user-upload/396c242edb0a-20220508.png)

# ログの構造化

Rails のログを JSON 形式にするための gem をインストールします。

https://github.com/reidmorrison/rails_semantic_logger

~~~diff rb:Gemfile
+ gem "rails_semantic_logger"
~~~

~~~
docker-compose run --rm web bundle install
docker-compose build
~~~

インストールされたら、`SemanticLogger` の設定を行います。ひとまず dev 環境のみに設定しています。

~~~diff rb:config/environments/development.rb
    config.hosts << "app"
+   config.rails_semantic_logger.format = :json
+ 
+   # Log setting
+   config.log_level = :debug
+   config.log_tags = {
+     request_id: :request_id,
+     ip:         :remote_ip
+   }
+ 
+   if ENV["RAILS_LOG_TO_STDOUT"].present?
+     $stdout.sync = true
+     config.rails_semantic_logger.add_file_appender = false
+     config.semantic_logger.add_appender(io: $stdout, formatter: :json)
+   end
  end
~~~

ログをJSON形式にするのと、IP アドレスも記録するようにしています。他にも色々と設定できるので、ドキュメントを読むと面白いかもしれません。

https://logger.rocketjob.io/rails.html

ツイートの投稿に成功したらログを出力するようにしておきます。

~~~diff rb:app/controllers/tweets_controller.rb
    # POST /tweets or /tweets.json
    def create
      @tweet = Tweet.new(tweet_params)
  
      respond_to do |format|
        if @tweet.save
          format.html { redirect_to tweet_url(@tweet), notice: "Tweet was successfully created." }
          format.json { render :show, status: :created, location: @tweet }
+     　　logger = SemanticLogger['Tweet']
+         log = {
+           message: "tweet was created",
+           content: @tweet.content,
+           title: @tweet.title,
+           user: "test user"
+         }
+         logger.info log
        else
          format.html { render :new, status: :unprocessable_entity }
          format.json { render json: @tweet.errors, status: :unprocessable_entity }
        end
      end
    end
~~~

これでアプリ側の準備は完了です。

# Fluent Bit の導入

FireLens ではログの出力に Fluent Bit を使います。Fluent Bit でやることとしては、ログのパース => フィルタ => 分類分け => 出力 という流れになります。

![](https://storage.googleapis.com/zenn-user-upload/97c8d14efd01-20220515.png)

https://docs.fluentbit.io/manual/concepts/data-pipeline/input

まずは Dockerfile を用意します。

~~~dockerfile:deploy/development/firelens/Dockerfile
FROM amazon/aws-for-fluent-bit:latest

COPY deploy/development/firelens/fluent-bit-custom.conf /fluent-bit/custom.conf
COPY deploy/development/firelens/myparsers.conf /fluent-bit/myparsers.conf
COPY deploy/development/firelens/stream_processor.conf /fluent-bit/stream_processor.conf

RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime 
~~~

3つの設定ファイルを作成しておきます。`myparsers.conf` はログのパース形式について記述しています。

~~~ :deploy/development/firelens/myparsers.conf
[PARSER]
    Name         rails
    Format       json
    # Command       |  Decoder  | Field | Optional Action   |
    # ==============|===========|=======|===================|
    Decode_Field_As    escaped     log 
~~~

https://docs.fluentbit.io/manual/pipeline/parsers/configuring-parser

`stream_processor.conf` ではログのストリームを作成しています。全てのログを配信するストリーム、Tweet 関連のログを配信するストリームを作ります。

~~~ :deploy/development/firelens/stream_processor.conf
[STREAM_TASK]
    Name all
    Exec CREATE STREAM all WITH (tag='all-log') AS SELECT * FROM TAG:'*-firelens-*';
[STREAM_TASK]
    Name tweet
    Exec CREATE STREAM tweet WITH (tag='tweet-log') AS SELECT * FROM TAG:'*-firelens-*' WHERE name = 'Tweet'; 
~~~

https://docs.fluentbit.io/manual/stream-processing/getting-started/fluent-bit-sql#create-stream-statement

最後にフィルターと出力先の設定をします。

~~~ :deploy/development/firelens/fluent-bit-custom.conf
[SERVICE]
    Parsers_File /fluent-bit/myparsers.conf
    Streams_File /fluent-bit/stream_processor.conf

[FILTER]
    Name parser
    Match *-firelens-*
    Key_Name log
    Parser rails
    Reserve_Data false

[FILTER]
    Name nest
    Match tweet-log
    Operation lift
    Nested_under payload

[FILTER]
    Name nest
    Match tweet-log
    Operation lift
    Nested_under named_tags

[OUTPUT]
    Name   cloudwatch
    Match  tweet-log
    region ${AWS_REGION}
    log_group_name ${LOG_GROUP_NAME}
    log_stream_prefix from-fluentbit/
    auto_create_group true

[OUTPUT]
    Name   cloudwatch
    Match  all-log
    region ${AWS_REGION}
    log_group_name ${LOG_GROUP_NAME}
    log_stream_prefix from-fluentbit/
    auto_create_group true

[OUTPUT]
    Name s3
    Match  tweet-log
    region ${AWS_REGION}
    bucket ${LOG_BUCKET_NAME}
    total_file_size 1M
    upload_timeout 1m
    use_put_object On
    s3_key_format /fluent-bit-logs/$TAG/%Y/%m/%d/$UUID.gz
    compression gzip
    content_type application/gzip 
~~~

`[FILTER] Name parser` は先ほどの設定ファイルを読んで rails ログのパースを行っています。

`[FILTER] Name nest` ではネストされている項目を一段上にリフトしています。これは AWS Athena でログをクエリする際に、ネストされた項目があるとエラーになってしまうためです。

`[OUTPUT]` ではログの出力先の設定をしています。今回は CloudWatch Logs と S3 にアップロードしていますが、様々な媒体にアップロードが可能です。詳しい設定についてはドキュメントを参照ください。

https://docs.fluentbit.io/manual/pipeline/outputs



# TaskDefinition の更新

アプリケーションコンテナのログ設定をデフォルト設定から `awsfirelens` に書き換えて、

~~~diff yml:deploy/templates/cfn-task.yml
        - Name: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
          Image: !Sub ${AWS::AccountId}.dkr.ecr.ap-northeast-1.amazonaws.com/${ServiceName}${EnvSuffix}:latest
          LogConfiguration:
-           LogDriver: awslogs
-           Options:
-             awslogs-group: !Sub /aws/ecs/${ServiceName}${EnvSuffix}
-             awslogs-region: ap-northeast-1
-             awslogs-stream-prefix: ecs
+           LogDriver: awsfirelens
          PortMappings:
            - ContainerPort: 80
              HostPort: 80
~~~

そして fluent-bit コンテナを追加します。

~~~diff yml:deploy/templates/cfn-task.yml
              ValueFrom: !Sub /${ServiceName}/${EnvName}/DB_HOST
            - Name: DATABASE_PASSWORD
              ValueFrom: !ImportValue cfn-rds:SecretsManagerDBPasswordArn
+       - Name: !Join [ '', [ log-router, !Ref EnvSuffix ] ]
+         Image: !Sub ${AWS::AccountId}.dkr.ecr.ap-northeast-1.amazonaws.com/${ServiceName}${EnvSuffix}:log-router
+         Cpu: 64
+         MemoryReservation: 128
+         LogConfiguration:
+           LogDriver: awslogs
+           Options:
+             awslogs-group: !Sub /aws/ecs/${ServiceName}-firelens${EnvSuffix}
+             awslogs-region: "ap-northeast-1"
+             awslogs-stream-prefix: "firelens"
+         FirelensConfiguration:
+           Type: fluentbit
+           Options:
+             config-file-type: file
+             config-file-value: /fluent-bit/custom.conf
+         Environment:
+           - Name: APP_ID
+             Value: !Join [ '', [ !Ref ServiceName, -log-router, !Ref EnvSuffix ] ]
+           - Name: AWS_ACCOUNT_ID
+             Value: !Ref AWS::AccountId
+           - Name: AWS_REGION
+             Value: ap-northeast-1
+           - Name: LOG_BUCKET_NAME
+             Value: !Join [ '', [ !Ref ServiceName, -logs, !Ref EnvSuffix ] ]
+           - Name: LOG_GROUP_NAME
+             Value: !Sub /aws/ecs/${ServiceName}${EnvSuffix}
      Cpu: 256
      ExecutionRoleArn: !ImportValue cfn-iam:ECSTaskExecutionRole
      Family: !Join [ '', [ !Ref ServiceName, !Ref EnvSuffix ] ]
~~~

## コード全文
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
            LogDriver: awsfirelens
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
        - Name: !Join [ '', [ log-router, !Ref EnvSuffix ] ]
          Image: !Sub ${AWS::AccountId}.dkr.ecr.ap-northeast-1.amazonaws.com/${ServiceName}${EnvSuffix}:log-router
          Cpu: 64
          MemoryReservation: 128
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Sub /aws/ecs/${ServiceName}-firelens${EnvSuffix}
              awslogs-region: "ap-northeast-1"
              awslogs-stream-prefix: "firelens"
          FirelensConfiguration:
            Type: fluentbit
            Options:
              config-file-type: file
              config-file-value: /fluent-bit/custom.conf
          Environment:
            - Name: APP_ID
              Value: !Join [ '', [ !Ref ServiceName, -log-router, !Ref EnvSuffix ] ]
            - Name: AWS_ACCOUNT_ID
              Value: !Ref AWS::AccountId
            - Name: AWS_REGION
              Value: ap-northeast-1
            - Name: LOG_BUCKET_NAME
              Value: !Join [ '', [ !Ref ServiceName, -logs, !Ref EnvSuffix ] ]
            - Name: LOG_GROUP_NAME
              Value: !Sub /aws/ecs/${ServiceName}${EnvSuffix}
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

# IAM の更新

権限周りも書き換えます。

`ECSTaskRole` にアプリケーションログを出力するための権限を、

~~~diff yml:deploy/templates/cfn-iam.yml
                  - 'ssmmessages:OpenDataChannel'
                Resource:
                  - '*'
+       - PolicyName: ECSFireLensPolicy
+         PolicyDocument:
+           Version: 2012-10-17
+           Statement:
+             - Effect: Allow
+               Action:
+                 - 's3:AbortMultipartUpload'
+                 - 's3:GetBucketLocation'
+                 - 's3:GetObject'
+                 - 's3:ListBucket'
+                 - 's3:ListBucketMultipartUploads'
+                 - 's3:PutObject'
+               Resource:
+                 - !Sub arn:aws:s3:::${ServiceName}-logs${EnvSuffix}
+                 - !Sub arn:aws:s3:::${ServiceName}-logs${EnvSuffix}/*
+             - Effect: Allow
+               Action:
+                 - 'logs:CreateLogGroup'
+                 - 'logs:CreateLogStream'
+                 - 'logs:DescribeLogGroups'
+                 - 'logs:DescribeLogStreams'
+                 - 'logs:PutLogEvents'
+               Resource:
+                 - !Sub arn:aws:logs:ap-northeast-1:${AWS::AccountId}:log-group:/aws/ecs/${ServiceName}${EnvSuffix}:*
~~~

`ECSTaskExecutionRole` に firelens のログを出力するための権限を渡します。

~~~diff yml:deploy/templates/cfn-iam.yml
                  - 'secretsmanager:ListSecretVersionIds'
                Resource: !ImportValue cfn-rds:SecretsManagerDBPasswordArn
+       - PolicyName: ECSFireLensPolicy
+         PolicyDocument:
+           Version: 2012-10-17
+           Statement:
+             - Effect: Allow
+               Action:
+                 - 'logs:CreateLogGroup'
+                 - 'logs:CreateLogStream'
+                 - 'logs:DescribeLogGroups'
+                 - 'logs:DescribeLogStreams'
+                 - 'logs:PutLogEvents'
+               Resource:
+                 - !Sub arn:aws:logs:ap-northeast-1:${AWS::AccountId}:log-group:/aws/ecs/${ServiceName}-firelens${EnvSuffix}:*

  ECSCodeDeployRole:
    Type: 'AWS::IAM::Role'
~~~

## コード全文

:::message 
コピペする際は、`GitHubOrg` と `RepositoryName` の書き換え忘れにご注意ください。
:::

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
  GitHubOrg:
    Type: String
    Default: n1sym
  RepositoryName:
    Type: String
    Default: rails-cfn-hands-on
  
Resources:
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
        - PolicyName: ECSFireLensPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 's3:AbortMultipartUpload'
                  - 's3:GetBucketLocation'
                  - 's3:GetObject'
                  - 's3:ListBucket'
                  - 's3:ListBucketMultipartUploads'
                  - 's3:PutObject'
                Resource:
                  - !Sub arn:aws:s3:::${ServiceName}-logs${EnvSuffix}
                  - !Sub arn:aws:s3:::${ServiceName}-logs${EnvSuffix}/*
              - Effect: Allow
                Action:
                  - 'kms:Decrypt'
                  - 'kms:GenerateDataKey'
                Resource:
                  - '*'
              - Effect: Allow
                Action:
                  - 'logs:CreateLogGroup'
                  - 'logs:CreateLogStream'
                  - 'logs:DescribeLogGroups'
                  - 'logs:DescribeLogStreams'
                  - 'logs:PutLogEvents'
                Resource:
                  - !Sub arn:aws:logs:ap-northeast-1:${AWS::AccountId}:log-group:/aws/ecs/${ServiceName}${EnvSuffix}:*
              
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
        - PolicyName: ECSFireLensPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'logs:CreateLogGroup'
                  - 'logs:CreateLogStream'
                  - 'logs:DescribeLogGroups'
                  - 'logs:DescribeLogStreams'
                  - 'logs:PutLogEvents'
                Resource:
                  - !Sub arn:aws:logs:ap-northeast-1:${AWS::AccountId}:log-group:/aws/ecs/${ServiceName}-firelens${EnvSuffix}:*
  
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

  GithubOidc:
    Type: AWS::IAM::OIDCProvider
    Properties:
      Url: https://token.actions.githubusercontent.com
      ClientIdList: 
        - sts.amazonaws.com
      ThumbprintList:
        - 6938fd4d98bab03faadb97b34396831e3780aea1
  
  GitHubActionsRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Join [ '', [ github-oidc-, !Ref ServiceName, !Ref EnvSuffix ] ]
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
        - PolicyName: ECSCodeDeployPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - ecs:DescribeServices
                  - ecr:GetAuthorizationToken
                  - ecs:DescribeTaskDefinition
                  - ecs:RegisterTaskDefinition
                  - codedeploy:GetDeploymentGroup
                  - codedeploy:CreateDeployment
                  - codedeploy:GetDeployment
                  - codedeploy:GetDeploymentConfig
                  - codedeploy:RegisterApplicationRevision
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
                Resource: !Sub arn:aws:ecr:ap-northeast-1:${AWS::AccountId}:repository/${ServiceName}${EnvSuffix}
              - Effect: Allow
                Action:
                  - iam:PassRole
                Resource: 
                  - !GetAtt ECSTaskRole.Arn
                  - !GetAtt ECSTaskExecutionRole.Arn
  
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

# スタックの更新

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-iam.yml s3://cfn-sample-templates/services/cfn-sample/
aws s3 cp deploy/templates/cfn-task.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの更新

~~~
aws cloudformation update-stack --stack-name cfn-iam --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-iam.yml --capabilities CAPABILITY_NAMED_IAM
aws cloudformation update-stack --stack-name cfn-task --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-task.yml
~~~

# GitHub Actions の更新

ログコンテナもビルドするように更新します。

:::details .github/workflows/build.yml
~~~yml:.github/workflows/build.yml
name: code build
on: 
  workflow_dispatch:
  push:
    branches:
      - main
jobs:
  build:
    name: Build image
    env:
      SERVICE_NAME: cfn-sample-dev
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Cache App Docker layers
      uses: actions/cache@v2
      with:
        path: /tmp/.buildx-cache-app
        key: ${{ runner.os }}-buildx-${{ github.sha }}
        restore-keys: |
          ${{ runner.os }}-buildx-

    - name: Cache LogRouter Docker layers
      uses: actions/cache@v2
      with:
        path: /tmp/.buildx-cache-log
        key: ${{ runner.os }}-buildx-${{ github.sha }}
        restore-keys: |
          ${{ runner.os }}-buildx-

    - name: Configure AWS credentials from Test account
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-oidc-cfn-sample-dev
        aws-region: ap-northeast-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - uses: docker/build-push-action@v2
      id: build-app-image
      with:
        push: true
        file: deploy/development/Dockerfile
        tags: ${{ steps.login-ecr.outputs.registry }}/${{ env.SERVICE_NAME }}:latest
        cache-from: type=local,src=/tmp/.buildx-cache-app
        cache-to: type=local,dest=/tmp/.buildx-cache-app-new,mode=max
    
    - uses: docker/build-push-action@v2
      id: build-firelens-image
      with:
        push: true
        file: deploy/development/firelens/Dockerfile
        tags: ${{ steps.login-ecr.outputs.registry }}/${{ env.SERVICE_NAME }}:log-router
        cache-from: type=local,src=/tmp/.buildx-cache-log
        cache-to: type=local,dest=/tmp/.buildx-cache-log-new,mode=max

    - name: Move cache
      run: |
        rm -rf /tmp/.buildx-cache-app
        mv /tmp/.buildx-cache-app-new /tmp/.buildx-cache-app
        rm -rf /tmp/.buildx-cache-log
        mv /tmp/.buildx-cache-log-new /tmp/.buildx-cache-log
~~~
:::

:::details .github/workflows/deploy.yml
~~~yml:.github/workflows/deploy.yml
name: code deploy
on: 
  workflow_dispatch:
  push:
    branches:
      - 'feature/**'
jobs:
  test:
    name: Test
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/myapp_test
      DATABASE_PASSWORD: postgres
      DATABASE_USERNAME: postgres
      RAILS_ENV: test
      BUNDLE_PATH: vendor/bundle
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: myapp_test
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v3
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.1.2'
      - name: Cache gems
        uses: actions/cache@v1
        with:
          path: vendor/bundle
          key: ${{ runner.os }}-gem-${{ hashFiles('**/Gemfile.lock') }}
          restore-keys: |
            ${{ runner.os }}-gem-
      - name: Install dependencies
        run: bundle install
      - name: db setup
        run: bundle exec rake db:migrate
      - name: Run tests
        run: bundle exec rake test
  deploy:
    name: Deploy Fargate
    needs: test
    env:
      SERVICE_NAME: cfn-sample-dev
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Cache App Docker layers
      uses: actions/cache@v2
      with:
        path: /tmp/.buildx-cache-app
        key: ${{ runner.os }}-buildx-${{ github.sha }}
        restore-keys: |
          ${{ runner.os }}-buildx-

    - name: Cache LogRouter Docker layers
      uses: actions/cache@v2
      with:
        path: /tmp/.buildx-cache-log
        key: ${{ runner.os }}-buildx-${{ github.sha }}
        restore-keys: |
          ${{ runner.os }}-buildx-

    - name: Configure AWS credentials from Test account
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-oidc-${{ env.SERVICE_NAME }}
        aws-region: ap-northeast-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - uses: docker/build-push-action@v2
      id: build-app-image
      with:
        push: true
        file: deploy/development/Dockerfile
        tags: ${{ steps.login-ecr.outputs.registry }}/${{ env.SERVICE_NAME }}:latest
        cache-from: type=local,src=/tmp/.buildx-cache-app
        cache-to: type=local,dest=/tmp/.buildx-cache-app-new,mode=max
    
    - uses: docker/build-push-action@v2
      id: build-firelens-image
      with:
        push: true
        file: deploy/development/firelens/Dockerfile
        tags: ${{ steps.login-ecr.outputs.registry }}/${{ env.SERVICE_NAME }}:log-router
        cache-from: type=local,src=/tmp/.buildx-cache-log
        cache-to: type=local,dest=/tmp/.buildx-cache-log-new,mode=max

    - name: Move cache
      run: |
        rm -rf /tmp/.buildx-cache-app
        mv /tmp/.buildx-cache-app-new /tmp/.buildx-cache-app
        rm -rf /tmp/.buildx-cache-log
        mv /tmp/.buildx-cache-log-new /tmp/.buildx-cache-log

    - name: Download task definition
      run: |
        aws ecs describe-task-definition --task-definition $SERVICE_NAME --query "taskDefinition.{containerDefinitions: containerDefinitions, family: family, taskRoleArn: taskRoleArn, executionRoleArn: executionRoleArn, networkMode: networkMode, volumes: volumes, placementConstraints: placementConstraints, requiresCompatibilities: requiresCompatibilities, cpu: cpu, memory: memory}" > task-definition.json
    
    - name: Deploy to Amazon ECS
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: task-definition.json
        service: ${{ env.SERVICE_NAME }}
        cluster: ${{ env.SERVICE_NAME }}
        codedeploy-appspec: deploy/development/appspec.yml
        codedeploy-application: AppECS-${{ env.SERVICE_NAME }}
        codedeploy-deployment-group: DgpECS-${{ env.SERVICE_NAME }}
~~~
:::

# 動作確認

任意の feature ブランチを作って、GitHub に push します。

デプロイが完了したらアプリケーションを開いてツイートをします。

![](https://storage.googleapis.com/zenn-user-upload/b179df4083d5-20220516.png)

ログが出力されていることを確認してみます。

- アプリケーションログ
  - Cloudwatch Logs `/aws/ecs/cfn-sample-dev`
  - S3 `s3://cfn-sample-logs-dev/fluent-bit-logs/tweet-log/`
- firelens 関連のログ
  - Cloudwatch Logs `/aws/ecs/cfn-sample-firelens-dev`

:::message 
一旦中断する場合は、忘れずにリソースの削除をしておきましょう。
:::