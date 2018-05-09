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

## Confige `awscli`

```bash
> awscli configure
```

---

Head to the AWS Console and set up an _IAM role_ for deploying

<notes>
https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html
</notes>

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

_Fn::ImportValue_ imports values from other templates

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

## Alternatives to Cloudformation

Why does everyone hate Cloudformation

---

Troposphere, Terraform, Sceptre, Serverless, Gordon, Zappa, Chalice

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

```
AWSTemplateFormatVersion: "2010-09-09"
Description: "Create a multi-az VPC"

Parameters:

  CIDRPrefix:
    Description: "Prefix for a /16 VPC e.g. 10.10"
    Type: String
    Default: '10.10'
```

---

## The VPC

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

## Internet Gateway

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

## Public Subnets

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

## Private Subnets

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

## Gateway IP Addresses

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

## NAT Gateways

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

## Public Routes

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

## Private Routes

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

## Outputs

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
