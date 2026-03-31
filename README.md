# AWS API Gateway Dual REST Entry POC

This repository records a working POC deployed in AWS account profile `aws-4` in region `eu-west-2`.

## Goal

Build a minimal backend that returns a simple JSON response and expose it through:

1. A private `REST API` reachable only through an `execute-api` interface `VPC Endpoint`
2. A public `Regional REST API` protected by `AWS WAF`

Both APIs share the same backend:

`API Gateway REST API -> VPC Link v2 -> internal ALB -> ECS Fargate`

## Final Architecture

```text
Private callers in VPC
  -> execute-api VPCE
  -> Private REST API
  -> VPC Link v2
  -> internal ALB
  -> ECS Fargate service

Internet callers
  -> Public Regional REST API
  -> WAF
  -> VPC Link v2
  -> same internal ALB
  -> same ECS Fargate service
```

## Deployed Resources

### Shared backend

- Region: `eu-west-2`
- VPC: `vpc-0d2570affb01999a6`
- Subnets:
  - `subnet-07914d1141c60807d`
  - `subnet-0a8e30e98c75454f6`
  - `subnet-0d189e31c561a1543`
- ECS cluster: `hello-private-api-cluster`
- ECS service: `hello-private-api-live-svc`
- ECS task definition: `hello-private-api-task:2`
- Internal ALB: `hello-private-api-alb`
- ALB ARN: `arn:aws:elasticloadbalancing:eu-west-2:379810014062:loadbalancer/app/hello-private-api-alb/254a9dc8e6d4d6e2`
- Target group: `hello-private-api-tg`
- API Gateway VPC Link v2: `t1hdlw`

### Private entry

- Private REST API ID: `plv2jm0jg6`
- Stage: `private`
- Interface VPCE for `execute-api`: `vpce-0b4c1ed151a61c38d`
- Private API policy restricts access to `aws:SourceVpce = vpce-0b4c1ed151a61c38d`

### Public entry

- Public Regional REST API ID: `ebwa6et4if`
- Stage: `public`
- Invoke URL: `https://ebwa6et4if.execute-api.eu-west-2.amazonaws.com/public/hello`
- WAF Web ACL: `hello-public-rest-waf`
- WAF ARN: `arn:aws:wafv2:eu-west-2:379810014062:regional/webacl/hello-public-rest-waf/a0c799c6-584e-4a65-92e3-67e84adad865`

### Legacy intermediate resource

An HTTP API `8q1t25se5b` was created during the first pass while validating ALB + VPC Link behavior. It is not required for the final dual-REST design and can be cleaned up later.

## Backend Behavior

The ECS container returns a fixed JSON payload.

Example response:

```json
{"message": "hello world", "service": "hello-private-api", "path": "/hello"}
```

## Validation Performed

### Public REST API

Executed from outside the VPC:

```bash
curl -sS https://ebwa6et4if.execute-api.eu-west-2.amazonaws.com/public/hello
```

Observed response:

```json
{"message": "hello world", "service": "hello-private-api", "path": "/hello"}
```

### Private REST API

Validated from inside the same VPC using a temporary Fargate task that executed:

```python
import urllib.request
print(urllib.request.urlopen(
    "https://plv2jm0jg6.execute-api.eu-west-2.amazonaws.com/private/hello",
    timeout=20,
).read().decode())
```

Observed CloudWatch Logs output:

```json
{"message": "hello world", "service": "hello-private-api", "path": "/hello"}
```

## Important AWS Findings

### VPCE vs VPC Link

- `VPCE` is for clients inside a VPC to reach an AWS service privately
- `VPC Link` is for API Gateway to reach private resources inside a VPC

They solve different directions of traffic and are not interchangeable.

### Why this design used REST APIs

- `Private API` support in API Gateway is a `REST API` capability
- A public `Regional REST API` can independently attach `WAF`
- Both APIs can reuse the same backend integration

### What failed during implementation

1. The old `apigateway create-vpc-link` flow failed with ALB because it expected NLB-shaped input
2. This account could not create an additional NLB, returning:

```text
OperationNotPermitted: This AWS account currently does not support creating load balancers.
```

3. The final working solution used `VPC Link v2` and `REST API` integrations with:

- `connection-type = VPC_LINK`
- `connection-id = t1hdlw`
- `integration-target = <ALB load balancer ARN>`

## Console Navigation

### Public REST API

- AWS Console
- API Gateway
- REST APIs
- `hello-public-rest-api`
- Stages
- `public`

### Private REST API

- AWS Console
- API Gateway
- REST APIs
- `hello-private-rest-api`
- Stages
- `private`

### WAF

- AWS Console
- AWS WAF & Shield
- Web ACLs
- Region `eu-west-2`
- `hello-public-rest-waf`

### ECS / ALB

- ECS
- Cluster `hello-private-api-cluster`
- Service `hello-private-api-live-svc`

- EC2
- Load Balancers
- `hello-private-api-alb`

## Files Used During Build

- Deployment work directory: `/home/joseph_siyi/aws4-private-api-deploy`
- Main script: `/home/joseph_siyi/aws4-private-api-deploy/deploy.sh`

## Cleanup Targets

If this POC needs to be removed later, the main resources to delete are:

- REST APIs `plv2jm0jg6` and `ebwa6et4if`
- WAF `hello-public-rest-waf`
- VPC Link `t1hdlw`
- ECS service `hello-private-api-live-svc`
- ECS cluster `hello-private-api-cluster`
- ALB `hello-private-api-alb`
- Target group `hello-private-api-tg`
- VPCE `vpce-0b4c1ed151a61c38d`
- Optional legacy HTTP API `8q1t25se5b`
