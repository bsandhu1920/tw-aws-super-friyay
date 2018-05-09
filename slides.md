# TW AWS Superfriyay

an introduction to infrastructure

---

## What you'll do

- Deploy a serverless application to AWS Lambda
- Deploy a high availability and scalable application cluster to AWS EC2
- Deploy a group of Docker containers with Amazon ECS

---

## Things you need

- You should have an AWS account
- You should have the awscli installed on your machine
- You should have Docker installed on your machine
- If you add a domain name to your AWS account (it's in Route 53) you'll get to play with a couple of extra things


---

## Cloudformation

![cloudformation](/img/how-cloudformation-works.png)

----

#### template example using yaml

```
AWSTemplateFormatVersion: "2010-09-09"
Description: A sample template

Resources:
  MyEC2Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      ImageId: "ami-2f726546"
      InstanceType: t1.micro
      KeyName: testkey
      BlockDeviceMappings:
        -
          DeviceName: /dev/sdm
          Ebs:
            VolumeType: io1
            Iops: 200
            DeleteOnTermination: false
            VolumeSize: 20
```

----

#### Why use cloudformation?

- AWS default!
- no need for a "real" programming language
- a "blueprint" of your infrastructure as code
- can deploy newest services that Amazon supports

----

#### So why are there all the other options?
- They try to be "cloud agnostic"

----

#### Some alternatives
- Troposphere, Terraform, Sceptre
- FaaS specific: Serverless, Gordon, Zappa, Chalice
