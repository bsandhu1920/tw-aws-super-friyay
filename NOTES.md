
Introduction to Infrastructure

- What you'll do:
	- Deploy a serverless application to AWS Lambda
	- Deploy a high availability and scalable application cluster to AWS EC2
	- Deploy a group of Docker containers with Amazon ECS

Why it's important:

- Deploying to production is a core task of any application development team (especially a ThoughtWorks team)
- For a long time teams have prided themselves on things like "anyone on the team can deploy" through to "we deploy multiple times a day" and "our code goes to production automatically"
- Established cloud native companies ARE deploying multiple times a day, many deploy many times an hour
- Banks, large retailers, etc. are introducing cloud-style deployments for teams
	- Legal problems pending...
- A consultant developer in 2018 should know how to deploy something to "the Cloud"
- That's where we're starting... Let's deploy three small applications to Amazon.

Before you start:

- You should have an AWS account
- You should have the `awscli` installed on your machine
- If you add a domain name to your AWS account (it's in Route 53) you'll get to play with a couple of extra things

**Cloudformation**

- Groan
- CloudFormation supported JSON
	- This meant you'd have 3000 line JSON files, with no comments and strict rules around dangling commas
- Now it support YAML
	- This is better
	- But not that much better
- So why CF? Because it's Amazon's default! All the ops people at $client will know it, perhaps even prefer it. It doesn't require a "real" programming language and it can deploy the newest services that Amazon supports
- So why are there all the other options?
	- 1. They try to be "cloud agnostic"
	- 2. They let you program your infrastructure (let's not get started on what is code)
- Some alternatives worth looking at: Troposphere, Terraform, Sceptre
- FaaS specific: Serverless, Gordon, Zappa, Chalice

- Some examples of Cloudformation YAML


**Serverless**

- Serverless is a name for small pieces of code that run on servers
	- You don't need to _manage_ the servers 👍
	- You don't get to _manage_ the servers :-1:
- AWS Lambda, Azure Functions, Google Cloud Functions, OpenWhisk
- On AWS "serverless" means Lambda
- Lambda functions are executed when an event occurs
- Events can include:
	- HTTP Requests via API Gateway
	- An email received with Simple Email Service
	- A file being added to an S3 bucket
	- Scheduled CloudWatch Events
	- A whole bunch of other things
- Languages include Python, Node, C#, Java & Go

**Spamming a Slack room when something happens**

- A vpc
- Python function that checks an RSS feed
- CloudWatch event that executes it every 5 minutes
- DynamoDb to store the latest post
- Somewhere for our logs to go



**Virtual Private Cloud (VPC)**

- VPCs are the building block of your private cloud infrastructure
- Almost everything you launch will be launched into a VPC that you manage
- Some organisations create VPCs for each product, others create one VPC for each environment and deploy all products into that

- Anyone that's ever spoken to an AWS architect has probably seen the same network diagram: two private subnets, two public subnets, a gateway and some sort of bastion host to SSH in
	- That's how we do it in ap-southeast-2

- VPCs consist of a VPC, an internet gateway, public & private subnets, NAT gateways and route tables
- Does that sound annoying? Yep, it sure does! Let's step through the creation of one with Cloudformation

```
multi-az-vpc.yaml
```


**Our Lambda Function**

- ... Some python code with no dependencies
- We're cheating and including the Python code in our CF template
	- _Normally_ you'd use an S3 bucket
