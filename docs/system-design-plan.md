# CAS Platform — System Design Plan

**Status:** Iteration 2 — ready for final design review (2026-08-05)  
**Owner:** Mark Simon (`github.com/markgsimon`)  
**AWS account:** `391647542075` · **Region:** `us-east-1`  
**Monthly budget cap:** ≤ **$200** (all platform + observability)  
**Domain:** `mgsimon.com` (canonical origin)  
**Repo:** `github.com/markgsimon/cas-platform`

> Living document. Multiple chat contexts will refine this iteratively. Prefer updating this file over re-deriving context from scratch.

---

## 1. Overview (confirmed)

### Product intent
Personal computational / computer-algebra control plane exposed on the web.

**Phase 0 user journey (green thread):**
1. User visits `https://mgsimon.com`
2. Sees sign-in: two inputs (email, password) + button
3. On success, navigates to the mathematical software / **compute** page
4. Submits a basic arithmetic expression
5. HTTP request reaches FastAPI in AWS
6. Backend authenticates JWT → verifies user against DB (hashed password at login time; JWT for subsequent calls)
7. Parses expression, evaluates (`+ - * /` only), persists a `Calculation` row, returns result (+ stub steps)
8. React shows the result in the client

### Long-term shape (confirmed)
```
React SPA (https://mgsimon.com / S3 + CloudFront for www)
        │ HTTPS + JWT
        ▼
FastAPI control plane (auth, validation, history, orchestration)
        │
        ▼
PostgreSQL (RDS)
        │
        └──► Engine (Phase 0: safe Python arithmetic)
             later: NumPy/SymPy → C++ CAS / cheminformatics library
```

Backend owns auth, calculation history, request validation, and engine orchestration. The computation engine is deliberately swappable; the API contract stays stable.

### Success criterion (Phase 0)
From a cold browser on `mgsimon.com`: sign in → compute e.g. `2+3*4` → see correct result → row in Postgres → full resource lifecycle deployable via IaC + CI/CD. Also: run the backend **locally** against a local Postgres container (Rancher Desktop / Docker) so local dev emulates deploy as closely as practical.

### Cost check (Pricing Calculator)
EC2 + RDS Phase 0 sizing ≈ **~$23/mo** — within budget before Sentry/Grafana/S3/transfer headroom (cap $200).

---

## 2. Repository strategy (confirmed)

| Repo | Role | Lifecycle |
|------|------|-----------|
| `website` (existing) | React SPA static bundle | Cheap JS build → S3; keep distinct |
| `cas-platform` (this repo) | FastAPI backend + Terraform + backend/infra CI | Docker image, RDS, VPC, always-on compute |

**Decision:** Do **not** convert `website` into a monorepo now. Lifecycles differ (static hosting vs always-on API + DB).

**Website repo update (minor, required):**
- Keep existing CodePipeline path (`mgsimon.com` pipeline / `buildspec.yml`) if simplest.
- Frontend-related IaC (S3 / Route53 / CloudFront adoption) lives in **`cas-platform/infra`**, not in the website repo.
- Website build must support networking config: at minimum `REACT_APP_API_BASE_URL=https://api.mgsimon.com` (or equivalent) injected at build/deploy time.

Terraform itself is **free**; cost is only the AWS resources it manages. Importing existing frontend resources into state does not add a Terraform bill.

---

## 3. Existing AWS inventory (as of 2026-08-05)

### Route53 — hosted zone `mgsimon.com`
| Record | Type | Notes |
|--------|------|--------|
| `mgsimon.com` | A (alias) | → `s3-website-us-east-1.amazonaws.com` |
| `mgsimon.com` | NS / SOA | AWS defaults |
| `mgsimon.com` | TXT | Google site verification |
| `_e66bc21e6570bd054e828aba5f138f8f.www.mgsimon.com` | CNAME | ACM validation |

### CloudFront
- Distribution: `de3g26eyjfzj2.cloudfront.net`
- Alternate domain: **`www.mgsimon.com`** only (not apex)
- Custom cert: `www.mgsimon.com`, TLSv1.2_2021
- Apex `mgsimon.com` currently aliases **directly to S3 website endpoint** (not CloudFront)

### Canonical site origin (confirmed)
- **Canonical client origin:** `https://mgsimon.com` (what you type in the browser; apex A → S3 website).
- Route53 + CloudFront confirm a split today: apex → S3, `www` → CloudFront.
- **CORS / API browser origin for Phase 0:** allow `https://mgsimon.com`. Optionally also allow `https://www.mgsimon.com` if that hostname still serves the app; redirect-www-to-apex (or the reverse) can be a later hardening step.

### S3
- `mgsimon.com`
- `www.mgsimon.com`
- `codepipeline-us-east-1-99933166404` (pipeline artifact bucket)

### CodePipeline
- Name: `mgsimon.com`

### VPC
- **Greenfield VPC** for the CAS platform (do not reuse an existing VPC for this stack).

---

## 4. Infrastructure requirements

### 4.1 Target runtime architecture

```
Browser
  │
  ├─ https://mgsimon.com  →  S3 (www may hit CloudFront today)
  │
  └─ https://api.mgsimon.com
           │
           ▼
     EC2 (t4g.micro or t3.micro) + nginx (TLS) + Docker(FastAPI)
           │ SG: 5432 only from API SG
           ▼
     RDS PostgreSQL (db.t4g.micro, single-AZ, private subnet)
```

**Phase 0 choices (approved):**
| Layer | Choice |
|-------|--------|
| Compute | EC2 micro + nginx reverse proxy + Docker |
| DB | RDS PostgreSQL `db.t4g.micro`, single-AZ, ~20GB gp3, **not public** |
| Network | Greenfield VPC: public subnet (EC2) + private subnet(s) (RDS); **no NAT Gateway** initially |
| DNS | Route53 `api.mgsimon.com` → Elastic IP / instance |
| API TLS | **Let’s Encrypt on nginx** (Phase 0 default — see §4.7) |
| Secrets | **SSM Parameter Store** (DB URL, JWT secret, etc.) |
| Registry | **ECR** for FastAPI image |
| Frontend | Stay on known S3 (+ existing CloudFront for www); IaC in `cas-platform/infra` |
| CORS | Backend allows `https://mgsimon.com` |
| Observability | **Sentry** + **Grafana Cloud** (cheapest plans), within $200 total budget |
| Local DB access from laptop | Later: **Tailscale** (or similar) → private RDS; not public 5432 |

**NAT:** Forgo NAT Gateway now. Adding later is a moderate Terraform/network change — acceptable deferral.

**ALB:** Skip in Phase 0 (cost). Single-instance nginx is enough.

### 4.2 Networking must-haves
1. Internet → API: 443 only to nginx on EC2 (not Postgres).
2. API → RDS: SG allows 5432 **only** from API compute SG.
3. No public RDS.
4. Browser → API over HTTPS at `api.mgsimon.com`.
5. CORS explicit on FastAPI for `https://mgsimon.com`.
6. Local API testing against deployed backend: deferred; document later (VPN or locked-down allowlist).

### 4.3 Terraform scope — green thread (confirmed)

All under **`cas-platform/infra`**:

**Bootstrap / shared**
- [ ] S3 + DynamoDB for Terraform remote state (new bucket)
- [ ] IAM roles for CodePipeline / CodeBuild (infra + app), least privilege

**Network**
- [ ] Greenfield VPC, public + private subnets, IGW, route tables
- [ ] Security groups: `api` (443/SSM), `rds` (5432 from api only)
- [ ] No NAT Gateway in Phase 0

**Data**
- [ ] RDS PostgreSQL micro, subnet group, parameter group, encrypted storage
- [ ] SSM parameters: DB credentials/URL, JWT signing secret, app config

**Compute / deploy**
- [ ] ECR repository
- [ ] EC2 instance + EIP, IAM instance profile (ECR pull, SSM read, CloudWatch optional)
- [ ] User-data / compose layout for nginx + app container + certbot/LE renewal
- [ ] Route53 record: `api.mgsimon.com`

**Frontend adoption (IaC, not click-ops)**
- [ ] Import/manage relevant S3 / Route53 / CloudFront pieces from this repo’s `infra/`
- [ ] Path to set/verify API base URL for the React build

**Observability**
- [ ] Sentry DSN via SSM; Grafana Cloud ingest endpoint / agent config

**Outputs**
- API URL, ECR URL, RDS endpoint (sensitive), relevant SSM param names

### 4.4 Cost posture
- Pricing Calculator: EC2 + RDS ≈ **$23/mo** for chosen micros.
- Budget ceiling **$200/mo** inclusive of Sentry + Grafana Cloud and headroom.
- Avoid: Multi-AZ RDS, ALB, NAT Gateway, oversized instances — until needed.

### 4.5 Free Tier note
Account resources date to **2022**; assume Free Tier for EC2/RDS **expired**. Confirmed path: Billing → Free Tier; forward estimate via Pricing Calculator (~$23).

### 4.6 Workstation → RDS (later)
Phase 0: no public DB; ops via SSM on EC2 or app-only access.  
Later: **Tailscale** on EC2 (or tiny subnet router) → route to private RDS → laptop `psql` / SQLStudio.

### 4.7 API TLS: Let’s Encrypt on nginx vs ACM

| | Let’s Encrypt on nginx (Phase 0 choice) | ACM (AWS Certificate Manager) |
|--|----------------------------------------|-------------------------------|
| Who issues the cert | Let’s Encrypt (public CA) | Amazon (public CA) |
| Where private key lives | On the EC2 host (nginx); you renew via certbot | Inside AWS; **not exportable** to install on nginx |
| Fits our EC2+nginx design? | **Yes** — terminate TLS on the instance | Poor fit without **ALB or CloudFront** in front (ACM attaches to those) |
| Ops | Free; automate renewal (cron/systemd + certbot) | Managed renewals; usually pay for ALB (~$16–22/mo) if used only for TLS |
| Trust model | Same idea: browser trusts a public CA; server proves identity with cert + private key | Same, but AWS holds the key |

**Clarification:** “Encrypt on nginx” does **not** mean a self-signed / homemade CA. Let’s Encrypt is still a third-party public certificate authority. The difference vs ACM is **where TLS terminates and who holds the private key** — on your nginx box (LE) vs on an AWS-managed front door (ACM on ALB/CloudFront).

**Phase 0 decision:** Let’s Encrypt on nginx. Revisit ACM if/when we add CloudFront or ALB in front of the API.

---

## 5. CI/CD requirements

### Pipeline A — Website (existing `mgsimon.com`)
- Prefer **reuse as-is** for build/deploy simplicity.
- Minor extensions only as needed: env for API base URL; frontend DNS/S3 ownership gradually via `cas-platform/infra`.

### Pipeline B — Infra (`cas-platform` / `infra/**`)
- Distinct pipeline:
  1. `terraform fmt` / `validate` / `plan` (PR)
  2. `terraform apply` (main; **manual approval** recommended)

### Pipeline C — Backend app (`cas-platform` / app)
1. Lint + **pytest**
2. Build Docker image → push **ECR**
3. Deploy to EC2 (SSM or compose pull/up)
4. **Alembic migrations** as deploy step **before** serving new traffic
5. Healthcheck smoke

### Deploy cutover: API Gateway / proxy?
**Not required for Phase 0.** nginx + container replace is enough. Later zero-downtime → ALB/ECS rolling, not API Gateway as the primary cutover tool.

---

## 6. Frontend business requirements (confirmed)

**Journey**
1. Sign-in page: email + password + submit
2. Navigate to compute / mathematical software page
3. Expression input + submit button
4. `POST` calculate with `Authorization: Bearer <jwt>`
5. Render result (and stub steps if returned)
6. Sign-out clears session

**Routes (minimal)**
- `/signin`
- Compute page (new route and/or evolve `/Math`)
- Existing content pages remain

**Functional**
- Unauthenticated compute → redirect to sign-in
- Loading / error states
- API base URL via build env → `https://api.mgsimon.com`
- History UI optional in Phase 0 (`GET /calculations` approved)

**Non-goals Phase 0:** polished design system, graphing, NL input, public registration.

---

## 7. Backend design requirements (confirmed)

### Stack
- FastAPI + Pydantic
- SQLModel / SQLAlchemy 2.x → PostgreSQL
- Alembic migrations (CI/CD + local)
- Thin routers; services for auth + engine
- Config via environment / SSM-mapped env
- **Local parity:** `docker compose` with local Postgres on Rancher Desktop/Docker

### Auth
- `POST /auth/login` → JWT (email + password)
- Protected routes: validate JWT → load user → reject if missing/inactive
- **No public register endpoint in Phase 0**

### Seed user (confirmed)
- A migration or seed may insert a **single User row** with email (and metadata) and **`hashed_password` NULL**.
- Operator sets the password hash **only via SQL** (or equivalent one-off DB update) from the workstation/ops path.
- **No password, plaintext, or hash** is stored in the seed script, git, or CI.
- Login must fail closed if `hashed_password` is NULL.

### Data model

**User**  
`id`, `email` (unique), `hashed_password` (**nullable** until ops sets it), `created_at`, `is_active` (recommended)

**Calculation** (source of truth for history/results/steps)  
`id`, `owner_id` → User  
`expression`, `input_type` (`math`)  
`result`, `result_type` (`numeric` | `error`)  
`status` (`pending` | `success` | `error`)  
`steps` (JSON array; stub OK)  
`engine_version`, timestamps  

No separate step table yet.

### Compute
- `POST /api/v1/calculate` `{ "expression": "..." }`
- Auth required; persist with `owner_id`
- Safe parser/evaluator: **only** `+ - * /` (parentheses as needed); **no** raw `eval`
- Return result + stub steps

### Skeleton endpoints
- `GET /health`
- `POST /auth/login`
- `POST /api/v1/calculate`
- `GET /api/v1/calculations` (ownership + history)

### Quality bar
- pytest: auth, parser (ops/precedence/invalid), ownership, NULL password rejected
- Dockerfile + env config
- Alembic initial migration
- Local compose path documented and working

### Observability (app)
- Sentry SDK on FastAPI + React (DSN from env/SSM)
- Grafana Cloud for log ingest / search (agent or vendor endpoint)

---

## 8. Answers to open operational questions

### Free Tier — how to check
Billing → **Free Tier**. Care about **EC2** and **RDS**. Forward cost: Pricing Calculator (~$23 for our micros).

### API Gateway for deploy cutover
Skip for Phase 0.

### Canonical origin — Route53 / CloudFront?
Yes. Inventory shows apex → S3, www → CloudFront. Browser habit + design choice → canonical **`https://mgsimon.com`**.

---

## 9. Phased delivery (agreed)

1. Bootstrap tfstate + pipeline IAM  
2. Greenfield VPC + SGs + RDS  
3. ECR + EC2 + SSM secrets + `api.mgsimon.com` + Let’s Encrypt/nginx  
4. Backend deploy pipeline (skeleton image) + Alembic on deploy  
5. Frontend env/CORS + IaC adoption of S3/DNS/CF from `cas-platform/infra`  
6. Sentry + Grafana Cloud wiring  
7. Smoke: health → login → calculate → row in DB  
8. (Later) Tailscale → private RDS from laptop  

**Principle:** Infra green thread up before deep application work.

---

## 10. Decision log

| Date | Decision |
|------|----------|
| 2026-08-05 | Separate repos: `website` + `cas-platform` |
| 2026-08-05 | Greenfield VPC; EC2+nginx; RDS private; no NAT Phase 0 |
| 2026-08-05 | SSM Parameter Store; ECR; CORS `https://mgsimon.com` |
| 2026-08-05 | Budget ceiling $200/mo incl. Sentry + Grafana |
| 2026-08-05 | Frontend stays S3/CF; add IaC so DNS/S3 not click-ops |
| 2026-08-05 | Local backend + local Postgres container required |
| 2026-08-05 | Later DB laptop access via Tailscale (not public RDS) |
| 2026-08-05 | No API Gateway for Phase 0 cutover |
| 2026-08-05 | EC2+RDS Pricing Calculator ≈ $23/mo |
| 2026-08-05 | Canonical origin `https://mgsimon.com` |
| 2026-08-05 | API TLS: Let’s Encrypt on nginx (not ACM/ALB for Phase 0) |
| 2026-08-05 | Grafana Cloud (not self-hosted) |
| 2026-08-05 | Seed user with nullable password; hash set only via SQL; no register endpoint |
| 2026-08-05 | All Terraform under `cas-platform/infra` |

---

## 11. Open items (post final review)

Resolved for design freeze pending your read-through:
- [x] GitHub repo `markgsimon/cas-platform`
- [x] Canonical URL `https://mgsimon.com`
- [x] TLS: Let’s Encrypt on nginx
- [x] Grafana Cloud
- [x] Seed user / nullable password / SQL-set hash
- [x] Infra home: `cas-platform/infra`

Remaining implementation choices (not blocking design approval):
- [ ] Terraform import vs careful adopt for existing S3/Route53/CloudFront
- [ ] Exact EC2 arch: `t4g.micro` vs `t3.micro` at apply time
- [ ] www → apex redirect hardening (optional)

---

## 12. Document history

| Rev | Date | Notes |
|-----|------|-------|
| 0 | 2026-08-05 | Initial architecture proposal from design chat |
| 1 | 2026-08-05 | Iteration 1: inventory, budget $200, observability, local Postgres, frontend IaC, Tailscale-later, pipelines A/B/C |
| 2 | 2026-08-05 | Iteration 2: ~$23 EC2+RDS, canonical apex, LE/nginx TLS, Grafana Cloud, SQL-only password seed, infra in-repo; repo bootstrap |
