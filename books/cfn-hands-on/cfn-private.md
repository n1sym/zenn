---
title: "おまけ：Private Subnet に ECS を置くパターン"
---

おまけとして、ECS を Private Subnet に置くパターンも作ってみます。

この構成では ECS はパブリックな IP を持ちません。なので、ECS と VPC 外にあるリソース間の通信を可能にするために VPC エンドポイントを作っています。

![](https://storage.googleapis.com/zenn-user-upload/6e1b579ed535-20220521.png)

# 各リソース

:::details deploy/templates/pattern-private/cfn-base.yml
~~~yml:deploy/templates/pattern-private/cfn-base.yml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  Vpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  # ECS用のプライベートサブネット
  PrivateSubnetECS1A:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      CidrBlock: 10.0.8.0/24
      AvailabilityZone: ap-northeast-1a
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName
        - Key: Type
          Value: Isolated
  
  PrivateSubnetECS1C:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      CidrBlock: 10.0.9.0/24
      AvailabilityZone: ap-northeast-1c
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName
        - Key: Type
          Value: Isolated

  PrivateRouteTableECS:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref Vpc
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  PrivateSubnetECS1ARouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTableECS
      SubnetId: !Ref PrivateSubnetECS1A

  PrivateSubnetECS1CRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTableECS
      SubnetId: !Ref PrivateSubnetECS1C

  ## DB用のプライベートサブネット
  PrivateSubnetDb1A:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      CidrBlock: 10.0.16.0/24
      AvailabilityZone: ap-northeast-1a
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName
        - Key: Type
          Value: Isolated

  PrivateSubnetDb1C:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      CidrBlock: 10.0.17.0/24
      AvailabilityZone: ap-northeast-1c
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName
        - Key: Type
          Value: Isolated

  PrivateRouteTableDb:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref Vpc
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  PrivateSubnetDb1ARouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTableDb
      SubnetId: !Ref PrivateSubnetDb1A

  PrivateSubnetDb1CRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTableDb
      SubnetId: !Ref PrivateSubnetDb1C

  ## Ingress用のパブリックサブネット
  PublicSubnet1A:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      CidrBlock: 10.0.0.0/24
      AvailabilityZone: ap-northeast-1a
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  PublicSubnet1C:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: ap-northeast-1c
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref Vpc
      InternetGatewayId: !Ref InternetGateway
  
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref Vpc
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  
  PublicSubnet1ARouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1A
      RouteTableId: !Ref PublicRouteTable
  
  PublicSubnet1CRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1C
      RouteTableId: !Ref PublicRouteTable

  ## VPCエンドポイント(Egress通信)用のプライベートサブネット
  PrivateSubnetEgress1A:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      CidrBlock: 10.0.248.0/24
      AvailabilityZone: ap-northeast-1a
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName
        - Key: Type
          Value: Isolated

  PrivateSubnetEgress1C:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      CidrBlock: 10.0.249.0/24
      AvailabilityZone: ap-northeast-1c
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName
        - Key: Type
          Value: Isolated

  # セキュリティグループ
  SecurityGroupIngress:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Ingress Allowed Ports
      VpcId: !Ref Vpc
      SecurityGroupEgress:
        - CidrIp: 0.0.0.0/0
          Description: Allow all outbound traffic by default
          IpProtocol: "-1"
      SecurityGroupIngress:
        - CidrIp: 0.0.0.0/0
          Description: from 0.0.0.0/0:80
          FromPort: 80
          IpProtocol: tcp
          ToPort: 80
        - CidrIpv6: ::/0
          Description: from ::/0:80
          FromPort: 80
          IpProtocol: tcp
          ToPort: 80
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  SecurityGroupECS:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security Group of ECS
      VpcId: !Ref Vpc
      SecurityGroupEgress:
        - CidrIp: 0.0.0.0/0
          Description: Allow all outbound traffic by default
          IpProtocol: "-1"
      SecurityGroupIngress:
        - IpProtocol: tcp
          Description: HTTP for Ingress
          FromPort: 80
          SourceSecurityGroupId:
            Fn::GetAtt:
              - SecurityGroupIngress
              - GroupId
          ToPort: 80
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName
  
  SecurityGroupDb:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security Group of database
      VpcId: !Ref Vpc
      SecurityGroupEgress:
        - CidrIp: 0.0.0.0/0
          Description: Allow all outbound traffic by default
          IpProtocol: "-1"
      SecurityGroupIngress:
        - IpProtocol: tcp
          Description: PostgreSQL protocol from ECS
          FromPort: 5432
          SourceSecurityGroupId:
            Fn::GetAtt:
              - SecurityGroupECS
              - GroupId
          ToPort: 5432
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName
  
  SecurityGroupEgress:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security Group of VPC Endpoint
      VpcId: !Ref Vpc
      SecurityGroupEgress:
        - CidrIp: 0.0.0.0/0
          Description: Allow all outbound traffic by default
          IpProtocol: "-1"
      SecurityGroupIngress:
        - IpProtocol: tcp
          Description: HTTPS for Internal
          FromPort: 443
          SourceSecurityGroupId:
            Fn::GetAtt:
              - SecurityGroupECS
              - GroupId
          ToPort: 443
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  ### VPC Endpoints
  ECRApiInterfaceEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ecr.api'
      VpcId: !Ref Vpc
      SubnetIds: 
        - !Ref PrivateSubnetEgress1A
        - !Ref PrivateSubnetEgress1C
      SecurityGroupIds:
        - !Ref SecurityGroupEgress

  ECRDkrInterfaceEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ecr.dkr'
      VpcId: !Ref Vpc
      SubnetIds: 
        - !Ref PrivateSubnetEgress1A
        - !Ref PrivateSubnetEgress1C
      SecurityGroupIds:
        - !Ref SecurityGroupEgress

  LogsInterfaceEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.logs'
      VpcId: !Ref Vpc
      SubnetIds: 
        - !Ref PrivateSubnetEgress1A
        - !Ref PrivateSubnetEgress1C
      SecurityGroupIds:
        - !Ref SecurityGroupEgress

  SecretsManagerInterfaceEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.secretsmanager'
      VpcId: !Ref Vpc
      SubnetIds: 
        - !Ref PrivateSubnetEgress1A
        - !Ref PrivateSubnetEgress1C
      SecurityGroupIds:
        - !Ref SecurityGroupEgress

  SsmInterfaceEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ssm'
      VpcId: !Ref Vpc
      SubnetIds: 
        - !Ref PrivateSubnetEgress1A
        - !Ref PrivateSubnetEgress1C
      SecurityGroupIds:
        - !Ref SecurityGroupEgress
  
  SsmmessagesInterfaceEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ssmmessages'
      VpcId: !Ref Vpc
      SubnetIds: 
        - !Ref PrivateSubnetEgress1A
        - !Ref PrivateSubnetEgress1C
      SecurityGroupIds:
        - !Ref SecurityGroupEgress

  EventsInterfaceEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.events'
      VpcId: !Ref Vpc
      SubnetIds: 
        - !Ref PrivateSubnetEgress1A
        - !Ref PrivateSubnetEgress1C
      SecurityGroupIds:
        - !Ref SecurityGroupEgress
  
  S3GatewayEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcEndpointType: Gateway
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal: '*'
            Action: '*'
            Resource: '*'
      RouteTableIds:
        - !Ref PrivateRouteTableECS
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.s3'
      VpcId: !Ref Vpc

Outputs:
  VpcId:
     Value: !Ref Vpc
     Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, VpcId ] ]
  SecurityGroupIngressId:
     Value: !GetAtt SecurityGroupIngress.GroupId
     Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, SecurityGroupIngressId ] ]
  SecurityGroupECSId:
     Value: !GetAtt SecurityGroupECS.GroupId
     Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, SecurityGroupECSId ] ]
  SecurityGroupDbId:
    Value: !GetAtt SecurityGroupDb.GroupId
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, SecurityGroupDbId ] ]
  PrivateSubnetDb1A:
    Value: !GetAtt PrivateSubnetDb1A.SubnetId
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, PrivateSubnetDb1A ] ]
  PrivateSubnetDb1C:
    Value: !GetAtt PrivateSubnetDb1C.SubnetId
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, PrivateSubnetDb1C ] ]
  PrivateSubnetECS1A:
    Value: !GetAtt PrivateSubnetECS1A.SubnetId
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, PrivateSubnetECS1A ] ]
  PrivateSubnetECS1C:
    Value: !GetAtt PrivateSubnetECS1C.SubnetId
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, PrivateSubnetECS1C ] ]
  PublicSubnetIngress1A:
    Value: !GetAtt PublicSubnet1A.SubnetId
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, PublicSubnetIngress1A ] ]
  PublicSubnetIngress1C:
    Value: !GetAtt PublicSubnet1C.SubnetId
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, PublicSubnetIngress1C ] ]
~~~
:::

:::details deploy/templates/pattern-private/cfn-ecs.yml
~~~yml:deploy/templates/pattern-private/cfn-ecs.yml
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
          AssignPublicIp: DISABLED
          SecurityGroups:
            - !ImportValue cfn-base:SecurityGroupECSId
          Subnets:
            - !ImportValue cfn-base:PrivateSubnetECS1A
            - !ImportValue cfn-base:PrivateSubnetECS1C
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

:::details deploy/templates/pattern-private/cfn-batch.yml
~~~yml:deploy/templates/pattern-private/cfn-batch.yml
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
                AssignPublicIp: DISABLED
                SecurityGroups: 
                  - !ImportValue cfn-base:SecurityGroupECSId
                Subnets:
                  - !ImportValue cfn-base:PrivateSubnetECS1A
                  - !ImportValue cfn-base:PrivateSubnetECS1C
            LaunchType: FARGATE
            PlatformVersion: LATEST
~~~
:::

# スタックの更新

ecs / batch スタックが作成されている場合は一旦削除しておきます。

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/pattern-private/cfn-base.yml s3://cfn-sample-templates/services/cfn-sample/pattern-private/
aws s3 cp deploy/templates/pattern-private/cfn-ecs.yml s3://cfn-sample-templates/services/cfn-sample/pattern-private/
aws s3 cp deploy/templates/pattern-private/cfn-batch.yml s3://cfn-sample-templates/services/cfn-sample/pattern-private/
~~~

#### スタックの作成

~~~
aws cloudformation update-stack --stack-name cfn-base --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/pattern-private/cfn-base.yml
aws cloudformation create-stack --stack-name cfn-ecs --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/pattern-private/cfn-ecs.yml
aws cloudformation create-stack --stack-name cfn-batch --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/pattern-private/cfn-batch.yml
~~~

:::message 
最後に、忘れずにリソースの削除をしておきましょう。
:::