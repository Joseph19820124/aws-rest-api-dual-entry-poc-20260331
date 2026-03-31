# AWS CLI Commands

This file captures the main CLI flow used to build the POC.

All commands assume:

```bash
export AWS_PROFILE=aws-4
export AWS_REGION=eu-west-2
```

## 1. Inspect base network

```bash
aws ec2 describe-vpcs --filters Name=isDefault,Values=true
aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-0d2570affb01999a6
aws ec2 describe-security-groups --filters Name=vpc-id,Values=vpc-0d2570affb01999a6
```

## 2. Create execute-api interface VPCE

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0d2570affb01999a6 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.eu-west-2.execute-api \
  --subnet-ids subnet-07914d1141c60807d subnet-0a8e30e98c75454f6 subnet-0d189e31c561a1543 \
  --security-group-ids <vpce-sg-id> \
  --private-dns-enabled
```

## 3. Create ECS backend

```bash
aws ecs create-cluster --cluster-name hello-private-api-cluster
aws logs create-log-group --log-group-name /ecs/hello-private-api
aws ecs register-task-definition --cli-input-json file://task-def.json
aws ecs create-service \
  --cluster hello-private-api-cluster \
  --service-name hello-private-api-live-svc \
  --task-definition hello-private-api-task:2 \
  --desired-count 1 \
  --launch-type FARGATE \
  --load-balancers targetGroupArn=<tg-arn>,containerName=app,containerPort=3000 \
  --network-configuration 'awsvpcConfiguration={subnets=[subnet-07914d1141c60807d,subnet-0a8e30e98c75454f6,subnet-0d189e31c561a1543],securityGroups=[<ecs-sg-id>],assignPublicIp=ENABLED}'
```

## 4. Create internal ALB

```bash
aws elbv2 create-load-balancer \
  --name hello-private-api-alb \
  --scheme internal \
  --type application \
  --subnets subnet-07914d1141c60807d subnet-0a8e30e98c75454f6 subnet-0d189e31c561a1543 \
  --security-groups <alb-sg-id>

aws elbv2 create-target-group \
  --name hello-private-api-tg \
  --protocol HTTP \
  --port 3000 \
  --target-type ip \
  --vpc-id vpc-0d2570affb01999a6 \
  --health-check-protocol HTTP \
  --health-check-path /hello \
  --matcher HttpCode=200

aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:eu-west-2:379810014062:loadbalancer/app/hello-private-api-alb/254a9dc8e6d4d6e2 \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:eu-west-2:379810014062:targetgroup/hello-private-api-tg/74ff1e36547cd700
```

## 5. Create API Gateway VPC Link v2

```bash
aws apigatewayv2 create-vpc-link \
  --name hello-private-api-http-vpclink \
  --subnet-ids subnet-07914d1141c60807d subnet-0a8e30e98c75454f6 subnet-0d189e31c561a1543 \
  --security-group-ids sg-00623179069b9980b
```

## 6. Create public Regional REST API

```bash
aws apigateway create-rest-api \
  --name hello-public-rest-api \
  --endpoint-configuration types=REGIONAL

aws apigateway create-resource \
  --rest-api-id ebwa6et4if \
  --parent-id djazn2a0z2 \
  --path-part hello

aws apigateway put-method \
  --rest-api-id ebwa6et4if \
  --resource-id v0sfwt \
  --http-method GET \
  --authorization-type NONE

aws apigateway put-integration \
  --rest-api-id ebwa6et4if \
  --resource-id v0sfwt \
  --http-method GET \
  --type HTTP_PROXY \
  --integration-http-method GET \
  --connection-type VPC_LINK \
  --connection-id t1hdlw \
  --uri http://internal-hello-private-api-alb-1346920429.eu-west-2.elb.amazonaws.com/hello \
  --integration-target arn:aws:elasticloadbalancing:eu-west-2:379810014062:loadbalancer/app/hello-private-api-alb/254a9dc8e6d4d6e2

aws apigateway create-deployment \
  --rest-api-id ebwa6et4if \
  --stage-name public
```

## 7. Create and attach WAF to public REST API

```bash
aws wafv2 create-web-acl \
  --name hello-public-rest-waf \
  --scope REGIONAL \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=hello-public-rest-waf \
  --rules '[{"Name":"AWS-AWSManagedRulesCommonRuleSet","Priority":0,"Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesCommonRuleSet"}},"OverrideAction":{"None":{}},"VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"AWSManagedRulesCommonRuleSet"}}]'

aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:eu-west-2:379810014062:regional/webacl/hello-public-rest-waf/a0c799c6-584e-4a65-92e3-67e84adad865 \
  --resource-arn arn:aws:apigateway:eu-west-2::/restapis/ebwa6et4if/stages/public
```

## 8. Create private REST API

```bash
aws apigateway create-rest-api \
  --name hello-private-rest-api \
  --endpoint-configuration types=PRIVATE,ipAddressType=dualstack,vpcEndpointIds=vpce-0b4c1ed151a61c38d \
  --policy file://private-api-policy.json

aws apigateway create-resource \
  --rest-api-id plv2jm0jg6 \
  --parent-id 9rrminxw79 \
  --path-part hello

aws apigateway put-method \
  --rest-api-id plv2jm0jg6 \
  --resource-id ne21aq \
  --http-method GET \
  --authorization-type NONE

aws apigateway put-integration \
  --rest-api-id plv2jm0jg6 \
  --resource-id ne21aq \
  --http-method GET \
  --type HTTP_PROXY \
  --integration-http-method GET \
  --connection-type VPC_LINK \
  --connection-id t1hdlw \
  --uri http://internal-hello-private-api-alb-1346920429.eu-west-2.elb.amazonaws.com/hello \
  --integration-target arn:aws:elasticloadbalancing:eu-west-2:379810014062:loadbalancer/app/hello-private-api-alb/254a9dc8e6d4d6e2

aws apigateway create-deployment \
  --rest-api-id plv2jm0jg6 \
  --stage-name private
```

## 9. Validate

### Public REST API

```bash
curl -sS https://ebwa6et4if.execute-api.eu-west-2.amazonaws.com/public/hello
```

### Private REST API from inside the VPC

Use an ECS task or another private workload inside the same VPC:

```python
import urllib.request
print(urllib.request.urlopen(
    "https://plv2jm0jg6.execute-api.eu-west-2.amazonaws.com/private/hello",
    timeout=20,
).read().decode())
```

## Notes

- For REST API + VPC Link v2, the working `integration-target` value was the `ALB load balancer ARN`
- The earlier attempt using a listener ARN failed for REST API
- The account also rejected creation of an additional NLB, so the final design reused the existing internal ALB
