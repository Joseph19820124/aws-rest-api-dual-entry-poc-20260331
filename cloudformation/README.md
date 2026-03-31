# CloudFormation

This directory contains a parameterized CloudFormation template for the POC.

## Files

- `template.yaml`
- `parameters.example.json`

## What the template creates

- Security groups for ALB, ECS, and execute-api VPCE
- Interface VPCE for `execute-api`
- ECS cluster, task execution role, task definition, and Fargate service
- Internal ALB, target group, and listener
- API Gateway `VPC Link v2`
- Public `Regional REST API`
- Public `WAFv2` Web ACL and association
- Private `REST API` with a resource policy restricted to the created VPCE

## Deploy

```bash
export AWS_PROFILE=aws-4
export AWS_REGION=eu-west-2

aws cloudformation deploy \
  --stack-name hello-private-api-dual-rest-poc \
  --template-file cloudformation/template.yaml \
  --parameter-overrides file://cloudformation/parameters.example.json \
  --capabilities CAPABILITY_NAMED_IAM
```

If your CLI does not accept JSON directly in `--parameter-overrides`, convert the values to inline pairs:

```bash
aws cloudformation deploy \
  --stack-name hello-private-api-dual-rest-poc \
  --template-file cloudformation/template.yaml \
  --parameter-overrides \
    ProjectName=hello-private-api \
    VpcId=vpc-0d2570affb01999a6 \
    VpcCidr=172.31.0.0/16 \
    SubnetIds=subnet-07914d1141c60807d,subnet-0a8e30e98c75454f6,subnet-0d189e31c561a1543 \
    ApiStagePublic=public \
    ApiStagePrivate=private \
  --capabilities CAPABILITY_NAMED_IAM
```

## Validate

After deployment, read outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name hello-private-api-dual-rest-poc \
  --query 'Stacks[0].Outputs'
```

Public API:

```bash
curl -sS https://<PublicRestApiId>.execute-api.eu-west-2.amazonaws.com/public/hello
```

Private API must be called from inside the VPC through the interface VPCE:

```python
import urllib.request
print(urllib.request.urlopen(
    "https://<PrivateRestApiId>.execute-api.eu-west-2.amazonaws.com/private/hello",
    timeout=20,
).read().decode())
```

## Notes

- The template uses the new REST API integration fields `ConnectionId`, `ConnectionType`, and `IntegrationTarget`
- For this POC, `IntegrationTarget` is set to the ALB ARN
- The template parameterizes the VPC and subnet selection so it can be reused in another test environment
