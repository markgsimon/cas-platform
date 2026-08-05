# infra

Terraform for the CAS platform in AWS account `391647542075`, region `us-east-1`.

## Intended scope (Phase 0)

- Remote state (S3 + DynamoDB lock)
- Greenfield VPC (public EC2 subnet, private RDS subnet; no NAT initially)
- Security groups, RDS PostgreSQL (`db.t4g.micro`), EC2 (`t4g.micro`/`t3.micro`) + nginx/Docker
- ECR, SSM Parameter Store, Route53 `api.mgsimon.com`
- Light adoption/import of existing `mgsimon.com` S3 / Route53 / CloudFront wiring so client↔API config is not click-ops

Modules and root stacks will be added after system design freeze. Do not apply from an empty tree yet.

## Conventions

- No secrets in git; use SSM Parameter Store
- `.tfvars` and state files are gitignored
- Pipeline B (planned): `fmt` → `validate` → `plan` → approved `apply`
