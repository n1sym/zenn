---
title: "Scheduled Task"
---

この章では ECS Scheduled Task を用いて、バッチ処理を作ります。

![](https://storage.googleapis.com/zenn-user-upload/03eb3b2bce18-20220508.png)

# バッチ処理の作成

簡単なバッチ処理を作っておきます。

~~~rb:app/models/batch/create_tweet.rb
class Batch::CreateTweet
  def execute
    Tweet.create(title: "hello", content: "created in batch")
    puts "The batch has been executed."
  end
end 
~~~

~~~rb:lib/tasks/sample/batch.rake
namespace :sample do
  namespace :batch do
    namespace :create_tweet do
      desc 'batch to check the behavior'
      task execute: :environment do
        Batch::CreateTweet.new.execute
      end
    end
  end
end 
~~~

試しに実行してみます。

~~~
docker-compose run --rm --no-deps web rake sample:batch:create_tweet:execute
~~~

![](https://storage.googleapis.com/zenn-user-upload/fcdd079a7e11-20220508.png)

レコードが成功されていたら OK です。任意の featureブランチを作って GitHub に push して自動デプロイしておきます。AWS CLI で ECR にイメージを push してタスクを手動で再起動するのでも大丈夫です。

# Scheduled Task

バッチリソースは `AWS::Events::Rule` で定義します。

`ScheduleExpression:` はバッチのスケジュールを cron 式で記述します。下の記述だと 
`cron(0 * * * ? *)` となっており、これは毎時0分にバッチが起動することを表します。ここのパラメータは自由に書きかえてもらって大丈夫です。

https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/events/ScheduledEvents.html#CronExpressions

バッチ処理を行うには、勿論 Rails アプリケーションを立ち上げる必要があります。今回は ECS で利用するために作った ECS Task を流用します。 `Input:` の `"containerOverrides"` に記述して、立ち上げ時に実行するコマンドを上書きしています。

~~~yml:deploy/templates/cfn-batch.yml
AWSTemplateFormatVersion: 2010-09-09
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
  SampleBatchScheduleExpression:
    Type: String
    Default: cron(0 * * * ? *)

Resources:
  SampleBatchRule:
    Type: AWS::Events::Rule
    Properties:
      Description: !Sub ${ServiceName}${EnvSuffix} sample batch rule
      Name: !Join [ '', [ sample-batch-, !Ref ServiceName, !Ref EnvSuffix ] ]
      ScheduleExpression: !Ref SampleBatchScheduleExpression
      Targets:
        - Id: sample-batch
          Arn: !ImportValue cfn-ecs:ECSClusterArn
          RoleArn: !ImportValue cfn-iam:ECSEventsRole
          Input: !Sub "{\"containerOverrides\": [{\"name\": \"${ServiceName}${EnvSuffix}\",\"command\": [\"bundle\", \"exec\", \"rake\", \"sample:batch:create_tweet:execute\"]}]}"
          EcsParameters:
            TaskDefinitionArn: !Sub '{{resolve:ssm:/${ServiceName}/${EnvName}/ECSTaskDefinitionArn}}'
            TaskCount: 1
            NetworkConfiguration:
              AwsVpcConfiguration:
                AssignPublicIp: ENABLED
                SecurityGroups: 
                  - !ImportValue cfn-base:SecurityGroupEcsId
                Subnets:
                  - !ImportValue cfn-base:PublicSubnetIngress1A
                  - !ImportValue cfn-base:PublicSubnetIngress1C
            LaunchType: FARGATE
            PlatformVersion: LATEST
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-events-rule.html

# スタックの作成

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-batch.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの作成

~~~
aws cloudformation create-stack --stack-name cfn-batch --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-batch.yml
~~~



# 動作確認

無事に実行されたら完了です。バッチログは CloudWatch Logs に保存されています。

![](https://storage.googleapis.com/zenn-user-upload/d6d6003ee63d-20220508.png)

:::message 
一旦中断する場合は、忘れずにリソースの削除をしておきましょう。
:::