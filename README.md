# AWS Demo for Super Friday

This is a talk introducing AWS and Cloudformation that was given at Thoughtworks Super Friday in Melbourne. The talk was prepared by [Kylie Sy](https://github.com/kksy) and [Brenton Cleeland](https://github.com/sesh).

There are several stacks provided as resources here:

- [A Multi-AZ VPC](templates/0001-multi-az-vpc.yaml) that's used as a base for the other stacks
- [An EC2 Cluster](templates/0003-ec2-cluster.yaml) set up inside an auto scaling group across multiple subnets
- [An ECR Repository Demo](templates/0004-ecr-repository.yaml)
- [A complete ECS Cluster running a single container](templates/0005-ecs-cluster.yaml)
- [A sample of configuring Route 53](templates/0006-route-53.yaml) to point at a Load Balancer

The slides from the talk are generated with [big](https://github.com/tmcw/big) and are available in the [slides](slides) directory.

---

_Things are a little rough around the edges, but that's okay._
