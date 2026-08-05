# cas-platform

Control plane for a personal computer algebra / scientific computing platform:

- **FastAPI** backend (auth, calculation history, orchestration)
- **PostgreSQL** on AWS RDS
- **Terraform** IaC under `infra/`
- React client remains in the separate [`website`](https://github.com/markgsimon) repo (`mgsimon.com`)

## Layout

```
cas-platform/
  docs/                 System design & living plan
  infra/                Terraform (VPC, RDS, EC2, ECR, DNS, frontend adoption)
  backend/              (forthcoming) FastAPI application
```

## Docs

Start here: [`docs/system-design-plan.md`](docs/system-design-plan.md)

## Status

**Phase 0 system design is complete** (see `docs/system-design-plan.md`). Next: implement infra, CI/CD, and backend skeleton.
