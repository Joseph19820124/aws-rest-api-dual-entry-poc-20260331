# Cleanup Guide

This file lists the main resources created during the POC and the usual deletion order.

## Important note

Delete dependent entry points before deleting shared backend resources.

Recommended order:

1. Public REST API
2. Private REST API
3. WAF association and WAF Web ACL
4. API Gateway VPC Link v2
5. ECS service
6. ECS cluster
7. ALB listener
8. ALB
9. Target groups
10. Interface VPCE
11. Optional legacy HTTP API
12. Temporary task definitions and log groups

## Resource IDs

- Public REST API: `ebwa6et4if`
- Private REST API: `plv2jm0jg6`
- WAF Web ACL: `hello-public-rest-waf`
- WAF ARN: `arn:aws:wafv2:eu-west-2:379810014062:regional/webacl/hello-public-rest-waf/a0c799c6-584e-4a65-92e3-67e84adad865`
- VPC Link v2: `t1hdlw`
- VPCE: `vpce-0b4c1ed151a61c38d`
- ECS cluster: `hello-private-api-cluster`
- ECS service: `hello-private-api-live-svc`
- ALB ARN: `arn:aws:elasticloadbalancing:eu-west-2:379810014062:loadbalancer/app/hello-private-api-alb/254a9dc8e6d4d6e2`
- Target group ARN: `arn:aws:elasticloadbalancing:eu-west-2:379810014062:targetgroup/hello-private-api-tg/74ff1e36547cd700`
- Optional HTTP API: `8q1t25se5b`

## Example CLI commands

### Delete API Gateway resources

```bash
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws apigateway delete-rest-api --rest-api-id ebwa6et4if
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws apigateway delete-rest-api --rest-api-id plv2jm0jg6
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws apigatewayv2 delete-api --api-id 8q1t25se5b
```

### Delete WAF

First fetch the current lock token:

```bash
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws wafv2 get-web-acl \
  --scope REGIONAL \
  --id a0c799c6-584e-4a65-92e3-67e84adad865 \
  --name hello-public-rest-waf
```

Then delete:

```bash
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws wafv2 delete-web-acl \
  --scope REGIONAL \
  --id a0c799c6-584e-4a65-92e3-67e84adad865 \
  --name hello-public-rest-waf \
  --lock-token <LOCK_TOKEN>
```

### Delete VPC Link v2

```bash
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws apigatewayv2 delete-vpc-link --vpc-link-id t1hdlw
```

### Delete ECS service and cluster

```bash
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws ecs delete-service \
  --cluster hello-private-api-cluster \
  --service hello-private-api-live-svc \
  --force

AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws ecs delete-cluster \
  --cluster hello-private-api-cluster
```

### Delete ALB and target groups

Delete the ALB first:

```bash
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws elbv2 delete-load-balancer \
  --load-balancer-arn arn:aws:elasticloadbalancing:eu-west-2:379810014062:loadbalancer/app/hello-private-api-alb/254a9dc8e6d4d6e2
```

Then delete target groups:

```bash
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws elbv2 delete-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:eu-west-2:379810014062:targetgroup/hello-private-api-tg/74ff1e36547cd700
```

### Delete the interface VPCE

```bash
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws ec2 delete-vpc-endpoints \
  --vpc-endpoint-ids vpce-0b4c1ed151a61c38d
```

### Delete temporary log groups

```bash
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws logs delete-log-group --log-group-name /ecs/hello-private-api
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws logs delete-log-group --log-group-name /ecs/private-api-curl
AWS_PROFILE=aws-4 AWS_REGION=eu-west-2 aws logs delete-log-group --log-group-name /ecs/private-api-probe
```
