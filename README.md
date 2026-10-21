# ECS on EC2 with EFS, ALB and Global Accelerator (CloudFormation)

CloudFormation template that builds an ECS cluster backed by an EC2 Auto Scaling group, a shared encrypted EFS file system mounted on every container instance, and a public ALB fronted by AWS Global Accelerator. It creates the infrastructure only: no task definitions or services.

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

Template: `cloudformation-ecs.yml`

```
Global Accelerator (TCP 80/443) --> ALB (80 -> 301 HTTPS, 443 -> fixed 404)
ECS cluster <-- Auto Scaling group (launch template, ECS-optimized AL2) --NFS 2049--> EFS (3 mount targets)
```

- ECS cluster named after `ProyectID`
- Launch template (ECS-optimized Amazon Linux 2 AMI, encrypted gp3 root, no public IP) and Auto Scaling group with CPU-based step scaling (up >75% for 2 min, down <70% for 10 min)
- User data: registers with the cluster, installs packages, ships Docker logs to the `dockerlogs` CloudWatch log group, looks up the EFS ID by name and mounts it under a hardcoded project-specific path in `/opt/`
- EFS (encrypted, elastic throughput, AWS Backup enabled) with mount targets in three subnets; security group allows NFS only from the container instances
- Public ALB: HTTP redirects to HTTPS; HTTPS listener (TLS 1.3 policy, ACM certificate) returns a fixed 404 until you add rules/targets
- Global Accelerator with a TCP listener on 80/443 pointing to the ALB
- IAM role/instance profile for the container instances
- Outputs: cluster, ALB DNS name, instance security group, EFS ID, Global Accelerator DNS name

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `ProyectID` | Project name used for the cluster and resource names | (required) |
| `KeyPair` | Existing EC2 key pair | `<KEY_PAIR_NAME>` |
| `VpcId` | Existing VPC | (required) |
| `LoadBalancerSubnets` | Public subnets for the ALB | (required) |
| `ECSSubnets` | Subnets for the container instances | (required) |
| `IntraSubnet1/2/3` | Subnets for the EFS mount targets | (required) |
| `ACMARN` | ACM certificate for the HTTPS listener | `<CERTIFICATE_ARN>` |
| `InstanceType` | Container instance type | `t3a.medium` |
| `ISize` | Root volume size (GiB) | `100` |
| `AmiID` | SSM parameter for the ECS-optimized AMI | `/aws/service/ecs/optimized-ami/amazon-linux-2/recommended/image_id` |
| `LaunchTemplateVersionNumber` | Launch template version used by the ASG | `1` |
| `healthCheckType` | ASG health check type (`EC2`/`ELB`) | `ELB` |
| `DesiredCapacity` / `MinSize` / `MaxSize` | ASG sizing | `2` / `2` / `4` |

## Deploy

```bash
aws cloudformation deploy \
  --stack-name <STACK_NAME> \
  --template-file cloudformation-ecs.yml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    ProyectID=<PROJECT_ID> KeyPair=<KEY_PAIR_NAME> VpcId=<VPC_ID> \
    LoadBalancerSubnets=<PUBLIC_SUBNET_1>,<PUBLIC_SUBNET_2> \
    ECSSubnets=<PRIVATE_SUBNET_1>,<PRIVATE_SUBNET_2> \
    IntraSubnet1=<SUBNET_A> IntraSubnet2=<SUBNET_B> IntraSubnet3=<SUBNET_C> \
    ACMARN=<CERTIFICATE_ARN>
```

## Security notes

- The instance role is very broad: `AmazonS3FullAccess`, `AmazonSSMFullAccess`, `CloudWatchFullAccess`, `AmazonElasticFileSystemFullAccess`, `AmazonEC2ContainerRegistryFullAccess`, CodeDeploy policies and an inline `sts:*` on `*`.
- The container instance security group allows 80/443 from `0.0.0.0/0` (instead of only from the ALB security group) and SSH from `10.0.0.0/8`.
- User data uses IMDSv1 (`169.254.169.254` without a token) and installs legacy packages (`python27-pip`, `awslogs`); the launch template does not enforce IMDSv2.
- The Auto Scaling group references `LaunchTemplate: !` (a stray YAML tag) and a fixed launch template version number.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name <STACK_NAME>
```

The `dockerlogs` log group is created by the instances, not the stack; delete it manually. AWS Backup recovery points of the EFS file system (if any were taken) remain in the backup vault.

## License

MIT — see [LICENSE](LICENSE).
