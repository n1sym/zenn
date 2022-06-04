---
title: "Network"
---

ここからは CloudFormation をやっていきます。

この章では VPC / サブネット / セキュリティグループなど、ネットワーク関連のリソースを作成して、以下のような環境が構築されることを目指します。

![](https://storage.googleapis.com/zenn-user-upload/47e80f8c099e-20220423.png)

# CloudFormation について

CloudFormation の基本的な情報についてはこの本では解説しませんが、以下を一読すれば、必要な前提知識は揃うと思います。（簡潔にまとまっていてオススメの記事です）

https://zenn.dev/soshimiyamoto/articles/3c624728438902

# ファイルの置き場所

CloudFormation の設定ファイルは `.yml` 形式で記述して、`deploy/templates` ディレクトリ配下に配置します。今回は `cfn-base.yml` を作成します。

~~~
.
├─ deploy
│   └─ templates
│       ├─ cfn-base.yml
...     ...
│       └─ cfn-***.yml
...
~~~

# VPC

最初に作るのは、ネットワークの基礎となる VPC です。

必須パラメータである `CidrBlock` は素直に `10.0.0.0/16` とします。

`Tags` の設定は任意なのですが、CloudFormationで作ったリソースであることを分かりやすくするために、`AWS::StackName` を参照して設定しています。


~~~yml:deploy/templates/cfn-base.yml
Resources:
  Vpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName
~~~

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html



# InternetGateway

VPC 内のリソースがインターネットと通信できるようにするために `InternetGateway` を作成して、`AttachGateway` で先ほど作った VPC と紐づけます。

~~~yml:deploy/templates/cfn-base.yml
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
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-internetgateway.html

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc-gateway-attachment.html

# PublicSubnet

パブリックサブネットを作成します。

AZ障害への備えとして、複数の AvailabilityZone にサブネットを作成しています。

~~~yml:deploy/templates/cfn-base.yml
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
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet.html




# RouteTable

ルートテーブルは、ネットワークの経路を設定するためのリソースです。これを使って、パブリックサブネットとインターネット間の通信ができるようにします。

まずは `PublicRouteTable` を作成。宛先として `0.0.0.0/0` が指定された場合に `InternetGateway` へルーティングするように `PublicRoute` で設定します。そして `SubnetRouteTableAssociation` で `PublicSubnet` と `PublicRouteTable` を関連付けて、サブネットごとに経路を制御します。

~~~yml:deploy/templates/cfn-base.yml
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
~~~

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-routetable.html

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route.html

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnetroutetableassociation.html

# PrivateSubnet

パブリックサブネットと同じような感じで、プライベートサブネットも作成しておきます。`PrivateRouteTable` では外部インターネットとの紐づけは行いません。

~~~yml:deploy/templates/cfn-base.yml
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

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref Vpc
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  PrivateSubnetDb1ARouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      SubnetId: !Ref PrivateSubnetDb1A

  PrivateSubnetDb1CRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      SubnetId: !Ref PrivateSubnetDb1C
~~~

# SecurityGroup

今回必要になるセキュリティグループは3つです。

![](https://storage.googleapis.com/zenn-user-upload/47e80f8c099e-20220423.png)

インバウンドルールは以下に示す必要最低限なものだけを許可します。

- Ingress Security Group は インターネットからの接続のみを許可
- ECS Security Group は Ingress Security Group からの接続のみを許可
- DB Security Group は ECS Security Group からの接続のみを許可

アウトバウンドルールは全て `0.0.0.0/0` を許可します。

~~~yml:deploy/templates/cfn-base.yml
  SecurityGroupIngress:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ECS Allowed Ports
      VpcId: !Ref Vpc
      SecurityGroupEgress:
        - CidrIp: 0.0.0.0/0
          Description: Security Group of Ingress
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
~~~

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html



# Outputs

他のテンプレートファイルから参照したい値を、`Outputs:` フィールドに書いておきます。

~~~yml:deploy/templates/cfn-base.yml
Outputs:
  VpcId:
     Value: !Ref Vpc
     Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, VpcId ] ]
  SecurityGroupIngressId:
     Value: !GetAtt SecurityGroupIngress.GroupId
     Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, SecurityGroupIngressId ] ]
  SecurityGroupEcsId:
     Value: !GetAtt SecurityGroupECS.GroupId
     Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, SecurityGroupEcsId ] ]
  SecurityGroupDbId:
    Value: !GetAtt  SecurityGroupDb.GroupId
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
  PublicSubnetIngress1A:
    Value: !GetAtt PublicSubnet1A.SubnetId
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, PublicSubnetIngress1A ] ]
  PublicSubnetIngress1C:
    Value: !GetAtt PublicSubnet1C.SubnetId
    Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, PublicSubnetIngress1C ] ]
~~~

# 完成コード

:::details deploy/templates/cfn-base.yml
~~~yml:deploy/templates/cfn-base.yml
AWSTemplateFormatVersion: '2010-09-09'

Resources:
  Vpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  # ECS用のパブリックサブネット
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

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref Vpc
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  PrivateSubnetDb1ARouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      SubnetId: !Ref PrivateSubnetDb1A

  PrivateSubnetDb1CRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      SubnetId: !Ref PrivateSubnetDb1C
  
  # セキュリティグループ
  SecurityGroupIngress:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security Group of Ingress
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

Outputs:
  VpcId:
     Value: !Ref Vpc
     Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, VpcId ] ]
  SecurityGroupIngressId:
     Value: !GetAtt SecurityGroupIngress.GroupId
     Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, SecurityGroupIngressId ] ]
  SecurityGroupEcsId:
     Value: !GetAtt SecurityGroupECS.GroupId
     Export:
      Name: !Join [ ":", [ !Ref AWS::StackName, SecurityGroupEcsId ] ]
  SecurityGroupDbId:
    Value: !GetAtt  SecurityGroupDb.GroupId
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

# スタックの作成

#### テンプレートファイルを置いておくための s3 バケットを作成

~~~
aws s3 mb s3://cfn-sample-templates
~~~

#### テンプレートファイルをS3にアップロード
~~~
aws s3 cp deploy/templates/cfn-base.yml s3://cfn-sample-templates/services/cfn-sample/
~~~

#### スタックの作成

~~~
aws cloudformation create-stack --stack-name cfn-base --template-url https://cfn-sample-templates.s3.ap-northeast-1.amazonaws.com/services/cfn-sample/cfn-base.yml
~~~

CloudFormation のコンソールで、順次作られていくリソースを眺めることができます。

![](https://storage.googleapis.com/zenn-user-upload/5b57da8bd6be-20220429.png)

無事に作成されました。

![](https://storage.googleapis.com/zenn-user-upload/48878a248019-20220429.png)
