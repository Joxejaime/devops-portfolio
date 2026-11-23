# Terraform: AWS deployment (ECS Fargate)

Deploys the gateway behind an Application Load Balancer:

```
Internet -> ALB (public subnets) -> Fargate tasks (private subnets, no public IP)
                                      |-> CloudWatch Logs
ECR (immutable tags, scan on push)    |-> CloudWatch alarms -> SNS
```

Highlights: private tasks, least-privilege security groups (tasks accept traffic only from the
ALB), immutable image tags, ECS deployment circuit breaker with automatic rollback, CPU
target-tracking autoscaling, and CloudWatch alarms for 5xx, unhealthy hosts and p95 latency.

## Status: validated in CI, planned for Juega App once it reaches 5,000 users

CI runs the two checks below on every push. The stack is not applied yet: it is planned for Juega App once it reaches 5,000 users,
when the single server moves behind ECS Fargate and an ALB. Its patterns (OIDC, alarms, least-privilege groups) already
run there through 02.

```bash
terraform fmt -check -recursive
terraform init -backend=false && terraform validate
```

## Deploy

The service needs an image in ECR before it can start, so create the repository first.

```bash
cp terraform.tfvars.example terraform.tfvars        # edit image_tag
terraform init
terraform apply -target=aws_ecr_repository.gateway  # 1. create the repository only

REPO=$(terraform output -raw ecr_repository_url)
aws ecr get-login-password | docker login --username AWS --password-stdin "${REPO%%/*}"
docker build -t "$REPO:0.1.0" .. && docker push "$REPO:0.1.0"   # 2. push the image

terraform apply                                     # 3. everything else
curl "http://$(terraform output -raw alb_dns_name)/readyz"
```

## Cost note

A NAT gateway (~$32/month plus data) and an ALB (~$16/month plus LCUs) are the main
fixed costs, so `terraform destroy` the dev environment when you are not using it.

## Known limitations (deliberate scope cuts)

- Single NAT gateway: an AZ outage in the NAT's zone cuts egress. Use one per AZ in prod.
- HTTP-only unless `certificate_arn` is set.
- No WAF and no VPC flow logs; both are sensible additions for a public endpoint.
- Admin / fault-injection routes are never enabled here (no `GATEWAY_ADMIN_TOKEN`).
