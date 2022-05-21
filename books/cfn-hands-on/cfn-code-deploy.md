---
title: "CodeDeploy / GitHub Actions"
---

この章では CodeDeploy と GitHub Actions を用いて、自動デプロイフローを作ります。

![](https://storage.googleapis.com/zenn-user-upload/56ece2eeb4af-20220505.png)



# CodeDeploy

Code Deploy のアプリケーションを作ります。

~~~yml:deploy/templates/cfn-code-deploy.yml
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
  CodeDeployApplication:
    Type: AWS::CodeDeploy::Application
    Properties:
      ApplicationName: !Join [ '', [ AppECS-, !Ref ServiceName, !Ref EnvSuffix ] ]
      ComputePlatform: ECS
      Tags:
        - Key: "CfnStackName"
          Value: !Ref AWS::StackName
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-codedeploy-application.html

## スタックの作成

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-code-deploy.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの作成

~~~
aws cloudformation create-stack --stack-name cfn-code-deploy --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-code-deploy.yml
~~~

# DeploymentGroup

ECS で blue/green デプロイをするためのデプロイメントグループは CloudFormation に対応していません。

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-codedeploy-deploymentgroup.html


なのでコンソールでボタンをポチポチして作成することにします。

CodeDeploy のアプリケーションを開いて、デプロイグループの作成ボタンを押します。

![](https://storage.googleapis.com/zenn-user-upload/72dd2de1fe72-20220507.png)

デプロイグループ名は、`DgpECS-cfn-sample-dev` と入力します。サービスロールは`ecs-code-deploy-cfn-sample-dev` を選択します。あとは画像の通りです。

![](https://storage.googleapis.com/zenn-user-upload/0ff6de222632-20220507.png)
![](https://storage.googleapis.com/zenn-user-upload/b77e3159dd88-20220507.png)
![](https://storage.googleapis.com/zenn-user-upload/06cf5450bd42-20220507.png)

これで CodeDeploy 側の準備は完了です。

（ECSリソースを作り直すたびに、この手順を行う必要があります。）

# IAM

GitHub Actions からAWSリソースを操作したいので、OpenID Connect (OIDC) を用いて認証を行います。OIDC については以下の記事が詳しいです。

https://zenn.dev/miyajan/articles/github-actions-support-openid-connect

[公式テンプレート](https://github.com/aws-actions/configure-aws-credentials#sample-iam-role-cloudformation-template)を参考に、`cfn-iam.yml`ファイルに追記していきます。

`Parameters:` に以下を追記します。`GitHubOrg` にはGitHubでのユーザー名を、`RepositoryName` には GitHub Actions を行いたいリポジトリの名前を入力します。

~~~yml:deploy/templates/cfn-iam.yml
  GitHubOrg:
    Type: String
    Default: n1sym
  RepositoryName:
    Type: String
    Default: rails-cfn-hands-on
~~~

続いて `Resources:` フィールドです。

`GithubOidc:` の `ThumbprintList:` は書き変わることが[あります](https://github.blog/changelog/2022-01-13-github-actions-update-on-oidc-based-deployments-to-aws/)。上記の公式テンプレートに倣えば基本は大丈夫そうですが、留意しておく必要はありそうです。

~~~yml:deploy/templates/cfn-iam.yml
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
                  - ecr:GetAuthorizationToken
                  - ecs:DescribeServices
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
~~~

GitHub Actions に渡すロールに設定する権限については、ざっくりと以下のような感じです。

- ECR の認証
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
- ECS タスクの更新
  - `ecs:*`
- blue/green デプロイのリクエスト
  - `codedeploy:*`

# 完成コード

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
~~~

#### スタックの更新

`update-stack` で更新分のみを反映させることが可能です。

~~~
aws cloudformation update-stack --stack-name cfn-iam --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-iam.yml --capabilities CAPABILITY_NAMED_IAM
~~~

# GitHub Actions

GitHub Actions の設定に移ります。設定ファイルは `.github/workflows/` 下に配置します。

## ビルド

まずはビルド処理から行います。

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
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-oidc-${{ env.SERVICE_NAME }}
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

`on:` ブロックでは、GitHub Actions の起動条件を設定します。`workflow_dispatch:` はUIからワークフローを手動実行できるようにするオプションです。

また、main ブランチに push されたら自動で job が実行されるようなっています。ここで main ブランチを指定しているのは、GitHub Actions のキャッシュには「スコープ」という概念があり、デフォルトブランチでキャッシュを残しておく必要があるためです。

つまり、この `build.yml` のワークフローは、キャッシュを残すために実行しています。

https://zenn.dev/mallowlabs/articles/github-actions-cache-scope



`jobs: build:` の処理については、[`docker/build-push-action@v2`](https://github.com/docker/build-push-action) という上手いことキャッシュを利用しながら build + push を行ってくれる GitHub Action に乗っかっている感じです。キャッシュ周りは [local-cache](https://github.com/docker/build-push-action/blob/master/docs/advanced/cache.md#local-cache) パターンのテンプレートをECR用に書き換えています。

また、`AWS_ACCOUNT_ID` については Secrets から読み込むようにしているので、リポジトリの Settings 欄から設定しておく必要があります。`New repository secret`で作成できます。 

![](https://storage.googleapis.com/zenn-user-upload/b82d7a653d72-20220410.png)


## テスト + デプロイ

長いですが、一旦全文を貼ります。

~~~yml:.github/workflows/deploy.yml
name: code test and deploy
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
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-oidc-${{ env.SERVICE_NAME }}
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

feature ブランチに push されたら自動で job が実行されるようなっており、feature ブランチを更新したときに自動で開発環境に反映させるような流れをイメージしています。

GitHub Actions で Rails の自動テストを行うために、Ruby 環境を用意します。

~~~yml
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.1.2'
~~~

また、テストにはデータベースが必要なので、`services:` で `postgres` を立ち上げています。`DATABASE_URL:` でデータベースと接続したいので、`config/database.yml` に以下を**追記します。**

~~~diff yml:config/database.yml
  test:
    <<: *default
+   url: <%= ENV['DATABASE_URL'].presence %>
    database: myapp_test
~~~

テスト実行手順は以下のような感じになります。

~~~yml
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
~~~

`bundle install` 時に キャッシュを利用するため、`actions/cache@v1` を利用しています。

テストファイルは Rails セットアップで `scaffold` した際に生成されたものを実行しています。

続いて `deploy` ステップに移ります。ビルド周りの処理は `build.yml` と同じです。

`name: Download task definition` からがデプロイ処理となります。CodeDeploy にデプロイのリクエストを送る際にタスク定義ファイルが必要になるので、AWS から JSON 形式でタスク定義を fetch してローカルに保存しています。不要なパラメータが混じると GitHub Actions のログに警告が表示されるので、長々としたクエリでパラメータを取捨しています。

~~~yml
    - name: Download task definition
      run: |
        aws ecs describe-task-definition --task-definition $SERVICE_NAME --query "taskDefinition.{containerDefinitions: containerDefinitions, family: family, taskRoleArn: taskRoleArn, executionRoleArn: executionRoleArn, networkMode: networkMode, volumes: volumes, placementConstraints: placementConstraints, requiresCompatibilities: requiresCompatibilities, cpu: cpu, memory: memory}" > task-definition.json
~~~

最後に code deploy にリクエストを送ります。

~~~yml
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

`appspec.yml` というファイルが必要になるので、事前に**作成しておきます**。

~~~yml:deploy/development/appspec.yml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "task-definition.json"
        LoadBalancerInfo:
          ContainerName: "cfn-sample-dev"
          ContainerPort: 80
~~~

テストフローのキャッシュを残すために、main ブランチで実行する `test.yml` も作っておきます。 

# 完成コード

::: details .github/workflows/test.yml
~~~yml:.github/workflows/test.yml
name: code test
on: 
  workflow_dispatch:
  push:
    branches:
      - main
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
~~~
:::

::: details .github/workflows/build.yml
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
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-oidc-cfn-sample-dev
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
:::

::: details .github/workflows/deploy.yml
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
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-oidc-${{ env.SERVICE_NAME }}
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

任意の feature ブランチを作って、GitHub に push してみましょう。

![](https://storage.googleapis.com/zenn-user-upload/9f0b7d8a0c08-20220521.png)

blue/green デプロイのステータスは CodeDeploy から確認できます。

![](https://storage.googleapis.com/zenn-user-upload/5f4d4780a8fc-20220507.png)

だいたい2~3分くらいで完了すると思います。

![](https://storage.googleapis.com/zenn-user-upload/ed7a53609fac-20220507.png)

これでダウンタイム無しでアプリケーションを更新することができます。

:::message 
一旦中断する場合は、忘れずにリソースの削除をしておきましょう。
:::