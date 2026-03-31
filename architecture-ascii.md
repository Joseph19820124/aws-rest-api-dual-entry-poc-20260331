# Architecture ASCII

## Final Layout

```text
                              +----------------------------------+
                              |            Internet              |
                              +----------------+-----------------+
                                               |
                                               v
                                  +---------------------------+
                                  | Public Regional REST API  |
                                  | API ID: ebwa6et4if        |
                                  | Stage: public             |
                                  +---------------------------+
                                               |
                                               v
                                  +---------------------------+
                                  | AWS WAF Web ACL           |
                                  | hello-public-rest-waf     |
                                  +---------------------------+
                                               |
                                               v
                                    +-----------------------+
                                    | API Gateway VPC Link  |
                                    | v2 ID: t1hdlw         |
                                    +-----------------------+
                                               |
                                               v
    +----------------------------------------------------------------------------------+
    |                                     VPC                                          |
    |                                                                                  |
    |  +-----------------------------------+                                           |
    |  | internal ALB                      |                                           |
    |  | hello-private-api-alb             |                                           |
    |  +----------------+------------------+                                           |
    |                   |                                                              |
    |                   v                                                              |
    |  +-----------------------------------+                                           |
    |  | ECS Fargate Service               |                                           |
    |  | hello-private-api-live-svc        |                                           |
    |  | returns hello world JSON          |                                           |
    |  +-----------------------------------+                                           |
    |                                                                                  |
    +----------------------------------------------------------------------------------+


    +----------------------------------------------------------------------------------+
    |                                     VPC                                          |
    |                                                                                  |
    |  Private callers                                                                 |
    |     |                                                                            |
    |     v                                                                            |
    |  +-----------------------------------+                                           |
    |  | Interface VPCE: execute-api       |                                           |
    |  | vpce-0b4c1ed151a61c38d            |                                           |
    |  +----------------+------------------+                                           |
    +-------------------|--------------------------------------------------------------+
                        |
                        v
             +---------------------------+
             | Private REST API          |
             | API ID: plv2jm0jg6        |
             | Stage: private            |
             | Resource policy restricts |
             | access to the VPCE        |
             +---------------------------+
                        |
                        v
             +---------------------------+
             | API Gateway VPC Link      |
             | v2 ID: t1hdlw             |
             +---------------------------+
                        |
                        v
                  same internal ALB
                        |
                        v
                  same ECS service
```

## Responsibility Split

```text
VPCE
- Gives private clients inside the VPC a private path to API Gateway

VPC Link v2
- Gives API Gateway a private path into the VPC backend

WAF
- Protects the public REST API stage

ALB
- Internal backend entry shared by both APIs

ECS
- Runs the hello world service
```
