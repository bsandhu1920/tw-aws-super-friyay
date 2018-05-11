# AWS Super Friyaaay

with _Kylie Sy_ and _Brenton Cleeland_

---

An intro to infrastructure on AWS

---

## What we'll do today

- Basics of Cloud Formation
- Create a VPC
- Deploy a group of EC2 instances
- Deploy a Docker container to an ECS Cluster

---

Why you should learn this stuff

---

Deploying to production is a core task of any application development team

---

Anyone on your team should be able to deploy new infrastructure

---

You should understand the underlying infrastructure

---

## What you need

- An AWS account
- The `awscli` installed
- `docker` installed
- (optional) a domain name in Route 53

---

Head to the AWS Console and set up an _IAM role_ for deploying

<notes>
https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html
</notes>

---

Create a _keypair_ while you're there too

---

## Configure `awscli`

```bash
> awscli configure
```

---

Cloudformation

---

It's the AWS default

---

It's not _that_ bad

---

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "A token description of your stack"

Parameters:
  # Parameters that can be configured when creating the stack

Resources:
  # Resources that will be created as part of the deployment

Outputs:
  # Values that are output (and exported) when complete
```

---

There are a bunch of Cloudformation "functions"

---

_!Sub_ allows you do do string substitution:

---

_!Ref_ lets you refer to other resources and parameters:

---

_!Select_ grabs items from lists, and _!GetAZs_ lists Availability Zones.

---

_!GetAtt_ accesses attributes of resources

---

_!ImportValue_ imports values from other templates

---

AWS populates some parameters that you can use:

- AWS::StackName
- AWS::Region
- AWS::AccountsId

---

Parameters

---

```yaml
RepositoryName:
  Type: String
  AllowedPattern: "[a-z0-9]+"
  Description: The name of your repository
  Default: demo
```

---

Resources

---

```yaml
PublicSubnetA:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    AvailabilityZone: !Select [0, !GetAZs '']
    CidrBlock: !Sub '${CIDRPrefix}.10.0/24'
    Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName} Public Subnet A
```

---

Outputs

---

```yaml
VPC:
  Description: VPC for the Stack
  Value: !Ref VPC
  Export:
    Name: !Sub ${AWS::StackName}::VPC
```

---

Alternatives to Cloudformation

---

Why does everyone hate Cloudformation?

---

Troposphere, Terraform, Sceptre, _Serverless_, Gordon, Zappa, Chalice

---

Your first VPC

---

Let's make a _multi-AZ_ network, with _public and private_ subnets.

---

Your VPC will contain:

- An Internet Gateway
- 2x Public Subnets
- 2x Private Subnets
- NAT Gateways
- A public Route Table
- Private Route Tables
- Routes for accessing the internet

---

Woah!

---

![](https://s3-ap-southeast-2.amazonaws.com/brntn-screenshots/base-vpc.jpg)

---

Cool! 😎

---

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Create a multi-az VPC"

Parameters:

  CIDRPrefix:
    Description: "Prefix for a /16 VPC e.g. 10.10"
    Type: String
    Default: '10.10'
```

---

Your _VPC_ is your own Virtual Private Cloud where you can deploy other services

---

```yaml
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: !Sub '${CIDRPrefix}.0.0/16'
    Tags:
      - Key: Name
        Value: !Ref AWS::StackName
```

---

The _Internet Gateway_ gives your VPC a path to the Internet

---

```yaml
InternetGateway:
  Type: AWS::EC2::InternetGateway
  Properties:
    Tags:
      - Key: Name
        Value: !Ref AWS::StackName

InternetGatewayAttachment:
  Type: AWS::EC2::VPCGatewayAttachment
  Properties:
    VpcId: !Ref VPC
    InternetGatewayId: !Ref InternetGateway
```

---

_Subnets_ are logical network subdivisions

---

```yaml
PublicSubnetA:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    AvailabilityZone: !Select [0, !GetAZs '']
    CidrBlock: !Sub '${CIDRPrefix}.10.0/24'
    MapPublicIpOnLaunch: true
    Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName} Public Subnet A

PublicSubnetB:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    AvailabilityZone: !Select [1, !GetAZs '']
    CidrBlock: !Sub '${CIDRPrefix}.20.0/24'
    MapPublicIpOnLaunch: true
    Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName} Public Subnet B
```

---

```yaml
PrivateSubnetA:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    AvailabilityZone: !Select [0, !GetAZs '']
    CidrBlock: !Sub '${CIDRPrefix}.11.0/24'
    Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName} Private Subnet A

PrivateSubnetB:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    AvailabilityZone: !Select [1, !GetAZs '']
    CidrBlock: !Sub '${CIDRPrefix}.21.0/24'
    Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName} Private Subnet B
```

---

_Elastic IP Addresses_ are just IP addresses reserved for your use

---

```yaml
NatGatewayEIPA:
  Type: AWS::EC2::EIP
  DependsOn: InternetGatewayAttachment
  Properties:
    Domain: vpc

NatGatewayEIPB:
  Type: AWS::EC2::EIP
  DependsOn: InternetGatewayAttachment
  Properties:
    Domain: vpc
```

---

Our _NAT Gateways_ allow our private subnets to access the Internet, but stops the Internet from accessing them

---

```yaml
NatGatewayA:
  Type: AWS::EC2::NatGateway
  Properties:
    AllocationId: !GetAtt NatGatewayEIPA.AllocationId
    SubnetId: !Ref PublicSubnetA

NatGatewayB:
  Type: AWS::EC2::NatGateway
  Properties:
    AllocationId: !GetAtt NatGatewayEIPB.AllocationId
    SubnetId: !Ref PublicSubnetB
```

---

Our public _RouteTable_ lets our public subnet route to the Internet Gateway

---

```yaml
PublicRouteTable:
  Type: AWS::EC2::RouteTable
  Properties:
    VpcId: !Ref VPC
    Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName} Public Route table

DefaultPublicRoute:
  Type: AWS::EC2::Route
  DependsOn: InternetGatewayAttachment
  Properties:
    RouteTableId: !Ref PublicRouteTable
    DestinationCidrBlock: 0.0.0.0/0
    GatewayId: !Ref InternetGateway

PublicSubnetARouteTableAssociation:
  Type: AWS::EC2::SubnetRouteTableAssociation
  Properties:
    RouteTableId: !Ref PublicRouteTable
    SubnetId: !Ref PublicSubnetA

PublicSubnetBRouteTableAssociation:
  Type: AWS::EC2::SubnetRouteTableAssociation
  Properties:
    RouteTableId: !Ref PublicRouteTable
    SubnetId: !Ref PublicSubnetB
```

---

Our private _Route Table_ gives our private subnets access to the NAT gateways

---

```yaml
PrivateRouteTableA:
  Type: AWS::EC2::RouteTable
  Properties:
    VpcId: !Ref VPC
    Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName} Private Route Table A

DefaultPrivateRouteA:
  Type: AWS::EC2::Route
  DependsOn: InternetGatewayAttachment
  Properties:
    RouteTableId: !Ref PrivateRouteTableA
    DestinationCidrBlock: 0.0.0.0/0
    NatGatewayId: !Ref NatGatewayA

PrivateSubnetARouteTableAssociation:
  Type: AWS::EC2::SubnetRouteTableAssociation
  Properties:
    RouteTableId: !Ref PrivateRouteTableA
    SubnetId: !Ref PrivateSubnetA

PrivateRouteTableB:
  Type: AWS::EC2::RouteTable
  Properties:
    VpcId: !Ref VPC
    Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName} Private Route Table B

DefaultPrivateRouteB:
  Type: AWS::EC2::Route
  DependsOn: InternetGatewayAttachment
  Properties:
    RouteTableId: !Ref PrivateRouteTableB
    DestinationCidrBlock: 0.0.0.0/0
    NatGatewayId: !Ref NatGatewayB

PrivateSubnetBRouteTableAssociation:
  Type: AWS::EC2::SubnetRouteTableAssociation
  Properties:
    RouteTableId: !Ref PrivateRouteTableB
    SubnetId: !Ref PrivateSubnetB
```

---

Finally, lets _output_ some things we'll need in other templates...

---

```yaml
Outputs:

  VPC:
    Description: VPC for the Stack
    Value: !Ref VPC
    Export:
      Name: !Sub ${AWS::StackName}::VPC

  PublicSubnetA:
    Description: Public Subnet A
    Value: !Ref PublicSubnetA
    Export:
      Name: !Sub ${AWS::StackName}::PublicSubnetA

  PrivateSubnetA:
    Description: Private Subnet A
    Value: !Ref PrivateSubnetA
    Export:
      Name: !Sub ${AWS::StackName}::PrivateSubnetA

  PublicSubnetB:
    Description: Public Subnet B
    Value: !Ref PublicSubnetB
    Export:
      Name: !Sub ${AWS::StackName}::PublicSubnetB

  PrivateSubnetB:
    Description: Private Subnet B
    Value: !Ref PrivateSubnetB
    Export:
      Name: !Sub ${AWS::StackName}::PrivateSubnetB
```

---

Now, head to the console and deploy it!

---

Congratulations! You've deployed your first VPC!

---

Coffee Break?

🍺🍺🍺

---

Welcome back!

👋

---

Let's deploy some _EC2 Instances_

---

Your deployment will contain:

- Security Groups
- Load Balancer
- Auto Scaling Group
- Launch Configuration

---

What, what? No EC2 instances?!

---

![](https://s3-ap-southeast-2.amazonaws.com/brntn-screenshots/vpc-with-asg.jpg)

---

The magic of _Auto Scaling Groups_

---

![](https://s3-ap-southeast-2.amazonaws.com/brntn-screenshots/vpc-with-ec2.jpg)

---

Cool! 😎

---

Lets set up some parameters

---

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Deploy a cluster of EC2 instances"

Parameters:

  VpcStackName:
    Description: The name of your VPC stack
    Type: String
    Default: tw-demo-vpc

  KeyName:
    Description: The name of a PEM file that is available
    Type: AWS::EC2::KeyPair::KeyName
```

---

Create a _Security Group_ that lets traffic in from the internet

---

```yaml
LoadBalancerSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for ALB
    VpcId:
      Fn::ImportValue: !Sub "${VpcStackName}::VPC"
    SecurityGroupIngress:
      - IpProtocol: "tcp"
        FromPort: 80
        ToPort: 80
        CidrIp: '0.0.0.0/0'
```

---

Create a _Security Group_ that lets in SSH traffic

---

```yaml
BastionHostSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Allow SSH from Anywhere
    VpcId:
      Fn::ImportValue: !Sub "${VpcStackName}::VPC"
    SecurityGroupIngress:
      - IpProtocol: "tcp"
        FromPort: "22"
        ToPort: "22"
        CidrIp: '0.0.0.0/0'
```

---

And another _Security Group_ to let in traffic from the LoadBalancer and Bastion

---

```yaml
InstanceSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Allow HTTP from Load Balancer and SSH from Bastion
    VpcId:
      Fn::ImportValue: !Sub "${VpcStackName}::VPC"
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        SourceSecurityGroupId: !Ref BastionHostSecurityGroup
```

---

Create a _Load Balancer_ that listens for HTTP traffic

---

```yaml
LoadBalancer:
  Type: AWS::ElasticLoadBalancing::LoadBalancer
  Properties:
    CrossZone: true
    Listeners:
      - LoadBalancerPort: 80
        InstancePort: 80
        Protocol: HTTP
    HealthCheck:
      Target: "HTTP:80/"
      HealthyThreshold: 3
      UnhealthyThreshold: 3
      Interval: 30
      Timeout: 5
    SecurityGroups:
      - !Ref LoadBalancerSecurityGroup
    Subnets:
      - Fn::ImportValue: !Sub "${VpcStackName}::PublicSubnetA"
      - Fn::ImportValue: !Sub "${VpcStackName}::PublicSubnetB"
```

---

An _Auto Scaling Group_ manages our EC2 instances

---

```yaml
ServerGroup:
  Type: AWS::AutoScaling::AutoScalingGroup
  Properties:
    LaunchConfigurationName: !Ref ServerLaunchConfiguration
    MinSize: 2
    MaxSize: 4
    LoadBalancerNames:
      - !Ref LoadBalancer
    VPCZoneIdentifier:
      - Fn::ImportValue: !Sub "${VpcStackName}::PrivateSubnetA"
      - Fn::ImportValue: !Sub "${VpcStackName}::PrivateSubnetB"
```

---

And the _Launch Configuration_ defines how the instances start

---

```yaml
ServerLaunchConfiguration:
  Type: AWS::AutoScaling::LaunchConfiguration
  Properties:
    KeyName: !Ref KeyName
    ImageId: ami-f0a06892
    SecurityGroups:
      - !Ref InstanceSecurityGroup
    InstanceType: t2.micro
    UserData:
      Fn::Base64: !Sub |
        #!/bin/bash -xe
        yum update -y
        yum update -y aws-cfn-bootstrap
        yum install -y nginx
        service nginx start
        curl https://basehtml.xyz > /usr/share/nginx/html/index.html

        /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource ServerGroup --region ${AWS::Region}
        /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource ServerGroup --region ${AWS::Region}
```

---

Holey Moley!

---

Run to the AWS console and deploy this stack!

---

Your first EC2 instances! How easy was that?

👍

---

Let's create an ECS Cluster

---

"cloud-native"

👩‍💻☁️👍

---

What is ECS?

---

Amazon EC2 Container Service

---

...

---

Firstly, let's create our ECR Repository

---

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Set up an ECR repository"

Parameters:

  RepositoryName:
    Type: String
    AllowedPattern: "(?:[a-z0-9]+(?:[._-][a-z0-9]+)*/)*[a-z0-9]+(?:[._-][a-z0-9]+)*"
    Description: The name of your repository (all letters must be lowercase letters, must start and end with a letter or number, can contain [._-/], slashes must have a letter or number on either side)
```

---

```yaml
Resources:

  DockerRepository:
    Type: "AWS::ECR::Repository"
    Properties:
      RepositoryName: !Ref RepositoryName
      RepositoryPolicyText:
        Version: "2012-10-17"
        Statement:
          - Sid: AllowPushPull
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action:
              - "ecr:GetDownloadUrlForLayer"
              - "ecr:BatchGetImage"
              - "ecr:BatchCheckLayerAvailability"
              - "ecr:PutImage"
              - "ecr:InitiateLayerUpload"
              - "ecr:UploadLayerPart"
              - "ecr:CompleteLayerUpload"
```

---

```yaml
Outputs:

  DockerRepositoryUrl:
    Description: URL for the Docker repository
    Value: !Sub ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${RepositoryName}
    Export:
      Name: !Sub ${AWS::StackName}::DockerRepositoryUrl
```

---

Upload the template in the console now. We need that export!

---

Now let's push our sample project to ECR

---

If you have an image/project ready, you can use that. Otherwise, here's a hello world node app that you can clone

https://github.com/kksy/docker-express-boilerplate

---

```bash
# build the project
# docker build -t docker-express .
> make build
# tag image (docker-express) to be pushed to the repository (express-demo) you've created
> docker tag docker-express:latest awsaccountid.dkr.ecr.ap-southeast-2.amazonaws.com/express-demo:latest
# login to ecr
> eval $(aws ecr get-login --no-include-email --region ap-southeast-2)
# push image
> docker push awsaccountid.dkr.ecr.ap-southeast-2.amazonaws.com/express-demo
```

---

Pushed? Cool!

---

On with the _Cluster_...

---

We have a bunch of _Parameters_ this time

---

```yaml
Parameters:

  VpcStackName:
    Description: The name of your VPC stack
    Type: String
    Default: tw-demo-vpc

  EcrStackName:
    Description: The name of your ECR stack
    Type: String
    Default: tw-demo-ecr

  KeyName:
    Description: The name of a PEM file that is available
    Type: AWS::EC2::KeyPair::KeyName
    Default: "brntn-sydney"

  ServiceName:
    Description: The name of your service
    Type: String
    Default: testServiceName

  ClusterName:
    Description: The name of your ECS cluster
    Type: String
    Default: demo-cluster
```

---

Our Instance will need an _IAM Role_ that gives it permissions for ECS

---

```yaml
InstanceRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: [ec2.amazonaws.com]
          Action: ['sts:AssumeRole']
    Path: /
    Policies:
      - PolicyName: "ec2-ecs-instance-role"
        PolicyDocument:
          Statement:
            - Effect: Allow
              Action: ['ecs:CreateCluster', 'ecs:DeregisterContainerInstance', 'ecs:DiscoverPollEndpoint',
                        'ecs:Poll', 'ecs:RegisterContainerInstance', 'ecs:StartTelemetrySession',
                        'ecs:Submit*', 'logs:CreateLogStream', 'logs:PutLogEvents', 'ecr:BatchGetImage',
                        'ecr:GetAuthorizationToken', 'ecr:BatchCheckLayerAvailability', 'ecr:GetDownloadUrlForLayer',
                        "logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents", "logs:DescribeLogStreams"]
              Resource: '*'
```

---

That Role needs a _IAM Profile_ so it can be assigned to our instance

---

```yaml
InstanceProfile:
  Type: AWS::IAM::InstanceProfile
  Properties:
    Path: /
    Roles:
      - !Ref InstanceRole
```

---

We need another _Role_ that gives our containers the permissions they need

---

```yaml
TaskRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: [ecs.amazonaws.com]
          Action: ['sts:AssumeRole']
    Path: /
    Policies:
      - PolicyName: "ecs-service-role"
        PolicyDocument:
          Statement:
            - Effect: Allow
              Action: ['elasticloadbalancing:DeregisterInstancesFromLoadBalancer', 'elasticloadbalancing:DeregisterTargets',
                        'elasticloadbalancing:Describe*', 'elasticloadbalancing:RegisterInstancesWithLoadBalancer',
                        'elasticloadbalancing:RegisterTargets', 'ec2:Describe*', 'ec2:AuthorizeSecurityGroupIngress']
              Resource: '*'
```

---

Cool! Let's create our _Cluster_

---

```yaml
EcsCluster:
  Type: AWS::ECS::Cluster
  Properties:
    ClusterName: !Ref ClusterName
```

---

Again with the _security groups_!

---

```yaml
LoadBalancerSecurityGroup:
  Type: "AWS::EC2::SecurityGroup"
  Properties:
    Tags:
      - Key: "Name"
        Value: !Ref "AWS::StackName"
    GroupDescription: "ECS Cluster ALB Group"
    VpcId:
      Fn::ImportValue: !Sub "${VpcStackName}::VPC"
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: '0.0.0.0/0'
```

---

```yaml
ClusterInstanceSecurityGroup:
  Type: "AWS::EC2::SecurityGroup"
  Properties:
    Tags:
      - Key: "Name"
        Value: !Ref "AWS::StackName"
    GroupDescription: "ECS Instance Security Group"
    VpcId:
      Fn::ImportValue: !Sub "${VpcStackName}::VPC"
    SecurityGroupIngress:
      - IpProtocol: "tcp"
        FromPort: "22"
        ToPort: "22"
        CidrIp: '0.0.0.0/0'
      - IpProtocol: tcp
        FromPort: "80"
        ToPort: "80"
        SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
      - IpProtocol: tcp
        FromPort: 31000
        ToPort: 65535
        SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
```

---

Our ECS _Launch Configuration_ is a little bit different

---

```yaml
ClusterLaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      KeyName: !Ref KeyName
      ImageId: ami-efda148d
      SecurityGroups:
        - !Ref ClusterInstanceSecurityGroup
      InstanceType: t2.micro
      IamInstanceProfile: !Ref InstanceProfile
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          yum update -y
          yum update -y aws-cfn-bootstrap

          echo ECS_CLUSTER=${EcsCluster} >> /etc/ecs/ecs.config
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource ClusterAutoScalingGroup --region ${AWS::Region}
```

---

But the _Auto Scaling Group_ is basically the same

---

```yaml
  ClusterAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    DependsOn:
      - LoadBalancer
    Properties:
      LaunchConfigurationName: !Ref ClusterLaunchConfiguration
      MinSize: 2
      MaxSize: 2
      VPCZoneIdentifier:
        - Fn::ImportValue: !Sub "${VpcStackName}::PrivateSubnetA"
        - Fn::ImportValue: !Sub "${VpcStackName}::PrivateSubnetB"

```

---

This time we're using an _Application Load Balancer_

---

```yaml
  LoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Ref AWS::StackName
      SecurityGroups:
        - !Ref LoadBalancerSecurityGroup
      Subnets:
        - Fn::ImportValue: !Sub "${VpcStackName}::PublicSubnetA"
        - Fn::ImportValue: !Sub "${VpcStackName}::PublicSubnetB"
```

---

ALBs are configured with _Listener_ and _Target Groups_

---

```yaml
  HttpListener:
    Type: "AWS::ElasticLoadBalancingV2::Listener"
    Properties:
      LoadBalancerArn: !Ref LoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - TargetGroupArn: !Ref LoadBalancerTargetGroup
          Type: forward
```

---

```yaml
  LoadBalancerTargetGroup:
    Type: "AWS::ElasticLoadBalancingV2::TargetGroup"
    DependsOn: LoadBalancer
    Properties:
      Tags:
        - Key: "Name"
          Value: !Ref "AWS::StackName"
      Port: 5000
      Protocol: HTTP
      VpcId:
        Fn::ImportValue: !Sub "${VpcStackName}::VPC"
```

---

The _Task Definition_ tells ECS how to run our container

---

```yaml
  DockerTaskDefinition:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      Family: "docker-task"
      ContainerDefinitions:
        - Name: "docker-task"
          Image:
            !Join
              - ":"
              - - Fn::ImportValue: !Sub "${EcrStackName}::DockerRepositoryUrl"
                - "latest"
          Memory: 256
          PortMappings:
            - HostPort: 0
              ContainerPort: 5000
          Essential: true
```

---

And the _Service_ tells ECS where to run the container

---

```yaml
  Service:
    Type: "AWS::ECS::Service"
    DependsOn:
      - ClusterAutoScalingGroup
    Properties:
      ServiceName: !Ref ServiceName
      Cluster: !Ref EcsCluster
      DesiredCount: 2
      TaskDefinition: !Ref DockerTaskDefinition

      Role: !Ref TaskRole
      LoadBalancers:
        - ContainerName: "docker-task"
          ContainerPort: 5000
          TargetGroupArn: !Ref LoadBalancerTargetGroup
```

---

That's our ECS Stack!

---

A couple of outputs for convenience

---

```yaml
Outputs:

  EcrLoadBalancerHostedZoneId:
    Description: "Hosted Zone Id for ALB"
    Value: !GetAtt LoadBalancer.CanonicalHostedZoneID
    Export:
      Name: !Sub ${AWS::StackName}::LoadBalancerHostedZoneId

  EcrLoadBalancerDNSName:
    Description: "Hosted Zone Id for ALB"
    Value: !GetAtt LoadBalancer.DNSName
    Export:
      Name: !Sub ${AWS::StackName}::LoadBalancerDNSName
```

---

And we're done! Congratulations!

👊
