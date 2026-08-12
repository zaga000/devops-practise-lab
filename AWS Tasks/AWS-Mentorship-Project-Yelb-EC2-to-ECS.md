# AWS Mentorship Final Project — Deploy a 3-Tier App to EC2, then Migrate to ECS

**Audience:** mentee studying AWS (junior, new to AWS / Terraform / CI-CD)
**Mentor:** Oleg
**Format:** self-paced, phased checklist. Work top to bottom. Don't skip the *Definition of Done* on each phase.
**Estimated effort:** ~5–7 weeks part-time (one phase ≈ a few evenings).

---

## 1. What you are building and why

You will take a real **three-tier web application** and run it on AWS **twice**:

1. **Phase A — Virtual machines.** Deploy it onto **EC2 instances in an Auto Scaling Group**, configured automatically with **user data**, behind a load balancer.
2. **Phase B — Containers.** Re-package the same app as Docker containers and run it on **Amazon ECS**, using **Cloud Map** for service discovery.

Along the way you add **logging, monitoring, and distributed tracing through CloudWatch and X-Ray**, run **load tests with Locust**, and watch **autoscaling react in real time** — first on EC2, then on ECS.

The goal isn't just "make it work." It's to *feel the difference* between running software on VMs versus containers, and to build the supporting muscles every AWS engineer needs: networking, IaC, CI/CD, observability, and cost discipline.

### Learning objectives

By the end you will be able to:

- Design a VPC with **public / private / database** subnet tiers.
- Write **Terraform** that is reusable across **multiple environments** (Dev and Prod).
- Build **GitHub Actions** pipelines that deploy to different AWS accounts using **OIDC** (no long-lived keys).
- Stand up an app on **EC2 with an Auto Scaling Group** using **user data** bootstrapping.
- Implement **CloudWatch logs, metrics, dashboards, and alarms**, plus **X-Ray tracing** via the **ADOT (AWS Distro for OpenTelemetry) collector**.
- Use **S3** with a **Gateway VPC endpoint** so private resources reach S3 without a NAT gateway.
- Containerize the app, push to **ECR**, and run it on **ECS** with **Cloud Map** and **Container Insights**.
- **Load-test with Locust** and prove that autoscaling works on both platforms.
- Tear everything down cleanly and keep the bill near **zero**.

---

## 2. The application: `yelb`

We use **yelb** — a small, well-known sample 3-tier app that is purpose-built for exactly this kind of "VMs → containers" demonstration.

Repo: https://github.com/mreferre/yelb

### Components (this is your "3 tiers" + a cache)

| Component | Tier | Tech | Port | Notes |
|---|---|---|---|---|
| `yelb-ui` | Web / presentation | Angular app served by **nginx** | 80 | Browser-facing. Proxies API calls to the appserver. |
| `yelb-appserver` | Application / logic | **Ruby (Sinatra)** | 4567 | Talks to Redis and Postgres. This is where we add tracing. |
| `redis-server` | Cache | Redis | 6379 | Stores page-view counter. |
| `yelb-db` | Database | **PostgreSQL** | 5432 | Stores votes. Lives in the **DB subnet**. |

This maps cleanly onto our subnet tiers: **UI → public/ALB**, **appserver + redis → private (app) subnet**, **Postgres → DB subnet**.

### Why yelb (and the X-Ray caveat — read this)

yelb is the canonical app for an **EC2-then-ECS** migration story, but it has **no built-in AWS X-Ray instrumentation**. That's intentional in this project: **adding tracing is one of your learning tasks**, not something you get for free.

You will add tracing with the **ADOT collector** (AWS's supported OpenTelemetry distribution):

- On **EC2**: run the ADOT collector as a service on the instance (installed via user data) and turn on **OpenTelemetry auto-instrumentation** for the Ruby/Sinatra appserver so spans flow to X-Ray.
- On **ECS**: run the ADOT collector as a **sidecar container** in the task; the app sends telemetry to `localhost` and the collector exports to X-Ray.

> If tracing instrumentation turns out to be too time-consuming, treat full end-to-end traces as a **stretch goal** and make sure CloudWatch logs + metrics + Container Insights are solid first. The ADOT collector and the X-Ray service map are the priority deliverables; deep per-request spans inside the Ruby code are the bonus.

---

## 3. Target architecture

### Phase A — EC2 (VMs)

```
                          Internet
                             │
                    ┌────────▼────────┐
                    │  ALB (public)   │   public subnets (2 AZs)
                    └───┬─────────┬───┘
                        │         │
        ┌───────────────▼───┐ ┌───▼──────────────┐   private "app" subnets (2 AZs)
        │ ASG: yelb-ui +    │ │ ASG: yelb-ui +   │
        │ appserver + redis │ │ appserver+redis  │   ← EC2, bootstrapped by user data
        │ + ADOT collector  │ │ + ADOT collector │
        └─────────┬─────────┘ └────────┬─────────┘
                  │                     │
                  └──────────┬──────────┘
                             │
                    ┌────────▼────────┐
                    │ PostgreSQL      │   DB subnets (2 AZs) — RDS (free tier) or
                    │ (yelb-db)       │   self-managed Postgres on an instance
                    └─────────────────┘

   S3  ◄──── Gateway VPC endpoint (free) ──── private subnets   (logs, artifacts, Locust results)
   CloudWatch (logs/metrics/alarms/dashboards)   |   X-Ray (via ADOT collector)
```

### Phase B — ECS (containers)

```
                          Internet
                             │
                    ┌────────▼────────┐
                    │  ALB (public)   │
                    └────────┬────────┘
                             │ (target: yelb-ui service)
        ┌────────────────────▼─────────────────────┐  ECS cluster (private subnets)
        │  ECS services (Cloud Map namespace yelb.local)            │
        │  yelb-ui  ──►  yelb-appserver  ──►  redis-server          │
        │                    │                                      │
        │   each task: app container + ADOT sidecar (→ X-Ray)       │
        └────────────────────┬─────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │ PostgreSQL      │  DB subnets (RDS free tier recommended)
                    └─────────────────┘

   Container Insights (cluster/service/task metrics)  |  S3 via Gateway endpoint
   CloudWatch Logs (awslogs driver)  |  X-Ray service map
```

**Service discovery:** in Phase B, `yelb-ui` finds `yelb-appserver` (and appserver finds `redis-server`) by DNS names registered in a **Cloud Map** namespace (e.g. `appserver.yelb.local`) — no hard-coded IPs.

---

## 4. AWS accounts & environments

You already have an **AWS Organization** with two accounts:

| Environment | Account | Purpose |
|---|---|---|
| **Dev** | existing Dev account | Daily work. Spin up, break, tear down freely. |
| **Prod** | existing Prod account | "Real" deploy. Changes only via the pipeline with approval. |

Conventions for this project:

- **Region:** `us-east-1` (broadest free-tier coverage). Keep everything in one region.
- **Environment selection in Terraform:** **Terraform workspaces** (`dev`, `prod`) + a `*.tfvars` file per environment. Same code, different inputs.
- **No long-lived AWS keys anywhere.** GitHub Actions assumes a role in each account via **OIDC**.
- **Tag everything:** `Project=yelb-mentorship`, `Environment=dev|prod`, `ManagedBy=terraform`, `Owner=<name>`. Tags make cleanup and cost tracking possible.

---

## 5. Cost guardrails (read before you build anything)

This is a learning project. The target spend is **a few dollars, or zero**. The biggest risk is leaving billable things running. Internalize these:

**Free (or free-tier) — use freely:**
- EC2 `t3.micro` / `t2.micro` — 750 hrs/month free for 12 months.
- **S3 Gateway VPC endpoint** — *free*. This is exactly why requirement #6 exists.
- ECS control plane, task scheduling, Cloud Map *when used by Service Connect* — no charge.
- RDS `db.t3.micro` single-AZ — 750 hrs/month free for 12 months.
- CloudWatch — generous free tier (logs ingestion, basic metrics, a few alarms/dashboards).

**NOT free — watch these closely:**
- **NAT Gateway** — ~\$0.045/hr **plus** data processing ≈ **\$32+/month each, always-on**. *This is the #1 surprise bill.* Mitigations below.
- **Application Load Balancer** — ~\$0.0225/hr + LCUs ≈ **\$16+/month**. Needed, but **delete it when not in use**.
- **Fargate** — no free tier. If you use ECS on Fargate it bills per vCPU/GB/second. We avoid this by running **ECS on the EC2 launch type** (reuses free-tier instances). Fargate is an optional alternative.
- **Interface VPC endpoints** — billed per hour + per GB. Use the **Gateway** endpoint for S3 (free); only add interface endpoints if you truly need them.
- Data transfer, X-Ray traces, Container Insights custom metrics — small, but non-zero at scale.

**NAT Gateway strategy for this project:**
- Use **one** NAT (not one per AZ) to halve cost, **or**
- Use a **NAT instance** on a `t3.micro` (free-tier eligible) instead of the managed NAT Gateway, **or**
- Only create the NAT while you're actively working and `terraform destroy` it after each session.
- Remember: S3 traffic does **not** need NAT once the **Gateway endpoint** exists.

**Golden rules:**
1. `terraform destroy` (or at least destroy the expensive bits — NAT, ALB, RDS) at the end of every working session in **Dev**.
2. Set an **AWS Budgets** alarm (e.g. \$5 and \$20) in **both** accounts on day one.
3. Never run Prod overnight unless you're demoing it.

---

## 6. Repository layout & tooling

One GitHub repo. Suggested structure:

```
yelb-aws-mentorship/
├── app/                      # yelb source (fork or vendored) + Dockerfiles
├── infra/
│   ├── modules/              # reusable TF modules: vpc, alb, asg, rds, ecs, observability...
│   ├── bootstrap/            # one-time: S3 state bucket + DynamoDB lock table per account
│   ├── env/
│   │   ├── dev.tfvars
│   │   └── prod.tfvars
│   └── main.tf / variables.tf / providers.tf / backend.tf
├── load-test/                # Locust files (locustfile.py) + deploy config
├── .github/workflows/        # CI/CD pipelines
└── docs/                     # diagrams, runbook, screenshots of dashboards/traces
```

- **IaC:** Terraform. Remote state in **S3** with a **DynamoDB** lock table (one per account).
- **CI/CD:** GitHub Actions, OIDC into Dev/Prod.
- **Containers:** Docker + Amazon ECR.
- **Load testing:** Locust.

---

# Phased checklist

Each phase has a **Goal**, a **Checklist**, a **Definition of Done (DoD)**, and a **Cost note**. Check the boxes as you go. Commit your Terraform after each phase.

---

## Phase 0 — Foundations

**Goal:** repo, remote state, and keyless pipeline access exist before any infrastructure is written.

**Checklist**
- [ ] Create the GitHub repo with the structure above; add a README and `.gitignore` (Terraform, `*.tfstate`, `.terraform/`).
- [ ] In **each** account, create the Terraform **state backend**: an S3 bucket (versioned, encrypted, public access blocked) + a DynamoDB lock table. (Do this with a small `bootstrap` Terraform config or by hand — it's the chicken-and-egg step.)
- [ ] Configure `backend.tf` to use S3 + DynamoDB, keyed per environment.
- [ ] Create the **GitHub OIDC identity provider** and an **IAM role** in each account that trusts your repo (scoped to branch/environment). Note the role ARNs.
- [ ] Set up **Terraform workspaces** (`dev`, `prod`) and `env/dev.tfvars` + `env/prod.tfvars` (region, CIDRs, instance sizes, env name).
- [ ] Turn on **AWS Budgets** alarms (\$5 / \$20) in both accounts.
- [ ] Decide and document the **tagging standard**.
> **Q for review:** add a Makefile wrapping `terraform plan/apply -var-file=env/dev.tfvars` etc., since config is split per environment? Proposing this as a Phase 0 tooling item, not gating Phase 1 DoD.

**DoD:** `terraform init` succeeds against the S3 backend in the Dev workspace; a trivial resource (e.g. an SSM parameter) can be applied and destroyed; budgets are live.

**Cost note:** S3 + DynamoDB state is effectively free. Budgets are free.

---

## Phase 1 — Networking (VPC, subnets, S3 Gateway endpoint)

**Goal:** a VPC with three subnet tiers across two AZs, plus free S3 access from private subnets.

**Checklist**
- [ ] Write a **`vpc` Terraform module**: 1 VPC, 2 AZs.
- [ ] **Public subnets** (2): for the ALB and (optionally) a NAT.
- [ ] **Private "app" subnets** (2): for EC2/ECS workloads. No public IPs.
- [ ] **DB subnets** (2): for Postgres/RDS. No internet route.
- [ ] Internet Gateway + public route table.
- [ ] NAT (managed **single** NAT *or* a NAT instance — see cost guardrails) + private route table. Make the NAT toggleable via a variable so you can destroy it between sessions.
- [ ] **S3 Gateway VPC endpoint** attached to the private + DB route tables (requirement #6).
- [ ] Security groups: ALB SG (80/443 from internet), app SG (80/4567 from ALB SG; 6379 internal), DB SG (5432 from app SG only).
> **Q for review:** where should SGs live — inside the `vpc` module, or a separate `security-groups` submodule? Leaning toward inside `vpc`, since ASG/ALB/RDS modules will consume SG ids via outputs.
- [ ] (Recommended) **VPC Flow Logs** → CloudWatch Logs or S3 (via the new Gateway endpoint) for later analysis.
> **Q for review:** Flow Logs — own file (`modules/vpc/flow-logs.tf`) instead of inline in `main.tf`? Destination — CloudWatch Logs or S3? Leaning S3 via the Gateway endpoint for cost, toggled with an `enable_flow_logs` variable.

**DoD:** `terraform apply` in Dev creates the VPC; an instance in a private subnet can `aws s3 ls` **without** a NAT route (proves the Gateway endpoint works); DB subnet has no route to the internet.

**Cost note:** Subnets/route tables/SGs/Gateway endpoint are free. NAT is the only billable item here — keep it small or off.

---

## Phase 2 — EC2 3-tier deployment with Auto Scaling

**Goal:** yelb running on EC2 in an ASG behind the ALB, bootstrapped entirely by user data.

**Checklist**
- [ ] Provision the **database tier**: RDS PostgreSQL `db.t3.micro` (free tier) in the DB subnets with a subnet group — *recommended* — **or** a self-managed Postgres container on an instance in the DB subnet (cheapest). Load the yelb schema.
- [ ] Write an **ASG + Launch Template** module: `t3.micro`, private app subnets, min/desired/max (e.g. 2/2/4).
- [ ] **User data script** that on boot: installs Docker, pulls/builds the yelb-ui, yelb-appserver, and redis containers (or installs them directly), wires the appserver to the RDS endpoint and redis, and starts them. (Pass config via user data / SSM Parameter Store, not hard-coded.)
> **Q for review:** attach an IAM instance profile with `AmazonSSMManagedInstanceCore` for Session Manager access, instead of opening SSH in the app SG? Avoids a bastion/key pair entirely.
- [ ] Create the **ALB** + target group (health check on the UI) + listener; register the ASG with the target group.
- [ ] Confirm the app is reachable through the ALB DNS name and votes/page-views work end to end.
- [ ] Add a **target-tracking scaling policy** on CPU (e.g. 50%) so the ASG can scale out under load (used in Phase 5).

**DoD:** Opening the ALB URL shows the yelb UI; voting persists (Postgres) and the page-view counter increments (Redis); terminating one instance is self-healed by the ASG.

**Cost note:** Instances are free-tier; ALB and RDS are billable — destroy when idle.

---

## Phase 3 — Observability on EC2 (CloudWatch + X-Ray)

**Goal:** logs, metrics, dashboards, alarms, and traces for the EC2 deployment.

**Checklist**
- [ ] Install the **CloudWatch agent** via user data: ship app + nginx logs to CloudWatch Logs, and custom metrics (memory, disk — these aren't in default EC2 metrics).
- [ ] Create a **CloudWatch dashboard**: ALB request count + latency + 5xx, ASG CPU, instance count, app error logs.
- [ ] Create **CloudWatch alarms**: high CPU (feeds autoscaling), ALB 5xx, unhealthy host count.
- [ ] Install and run the **ADOT collector** on the instances (via user data) configured with the **X-Ray exporter** + an IAM instance-profile policy allowing `xray:PutTraceSegments`, `xray:PutTelemetryRecords`, and `logs:PutLogEvents`.
- [ ] Enable **OpenTelemetry auto-instrumentation** for the Ruby/Sinatra appserver so request spans reach X-Ray. *(Stretch: add manual spans around the Postgres/Redis calls.)*
- [ ] Confirm a populated **X-Ray service map** (UI → appserver → db/redis).

**DoD:** Logs searchable in CloudWatch Logs Insights; dashboard shows live traffic; a test request appears as a trace in the X-Ray service map.

**Cost note:** CloudWatch + X-Ray usage here is well within free tier at test volumes.

---

## Phase 4 — CI/CD with GitHub Actions (multi-environment)

**Goal:** infrastructure and app deploy through pipelines, not from your laptop, with Dev/Prod separation.

**Checklist**
- [ ] **Terraform PR workflow:** on pull request → `terraform fmt -check`, `validate`, and `plan` (Dev workspace); post the plan as a PR comment.
- [ ] **Dev deploy workflow:** on merge to `main` → assume the Dev OIDC role → `terraform apply` in the **dev** workspace.
- [ ] **Prod deploy workflow:** on a Git **tag/release** (or manual `workflow_dispatch`) → assume the Prod OIDC role → `terraform apply` in the **prod** workspace, gated by a **GitHub Environment protection rule** (manual approval).
- [ ] Use **GitHub Environments** (`dev`, `prod`) to scope secrets/variables (role ARNs, region) and approvals.
- [ ] (After Phase 6) Add an **app build/push workflow**: build Docker images, tag with the Git SHA, push to **ECR**.
- [ ] Verify no AWS access keys exist anywhere — only OIDC role assumption.

**DoD:** A PR shows a Terraform plan automatically; merging deploys to Dev; cutting a release deploys to Prod **only after** you approve it.

**Cost note:** GitHub Actions minutes for public/standard repos are free or cheap; no AWS cost beyond what the apply creates.

---

## Phase 5 — Load testing with Locust + EC2 autoscaling proof

**Goal:** drive traffic with Locust and watch the EC2 ASG scale out and back in.

**Checklist**
- [ ] Write a `locustfile.py` that exercises the yelb API (e.g. `GET` the restaurant list / votes endpoints and `POST` votes) through the ALB.
- [ ] Deploy **Locust** (master + a worker or two) — simplest is a single `t3.micro` EC2 in the public subnet, or a small container. Parameterize the target ALB URL.
- [ ] Write the **load results to S3** (through the Gateway endpoint) for later review.
- [ ] **Run a ramped load test**; watch the CloudWatch dashboard: CPU rises → alarm fires → **ASG adds instances**; stop load → instances scale back in.
- [ ] Capture screenshots/notes of the scale-out and scale-in events for the project writeup.

**DoD:** Documented evidence that increasing Locust load caused the ASG instance count to increase, and that it returned to baseline afterward.

**Cost note:** Run load tests in short bursts; tear down the Locust host afterward.

---

## Phase 6 — Containerize & push to ECR

**Goal:** the same app as versioned container images in your registry.

**Checklist**
- [ ] Confirm/author **Dockerfiles** for `yelb-ui`, `yelb-appserver` (yelb already ships images; build your own so you control them and can add the OTel layer).
- [ ] Create **ECR repositories** (Terraform) for each image with a lifecycle policy (keep last N images).
- [ ] Build images locally, tag with the Git SHA, push to ECR (then automate via the Phase 4 build workflow).
- [ ] Run the stack locally with `docker compose` to confirm parity before touching ECS.

**DoD:** Tagged images for ui and appserver exist in ECR; `docker compose up` runs the full app locally.

**Cost note:** ECR storage for a few small images is within free tier (500 MB/month for 12 months).

---

## Phase 7 — Deploy to ECS with Cloud Map

**Goal:** run yelb on ECS with DNS-based service discovery — the containerized counterpart of Phase 2.

**Checklist**
- [ ] Create an **ECS cluster** using the **EC2 launch type** (capacity provider backed by a `t3.micro` ASG) to stay in free tier. *(Fargate is the optional, simpler-but-billable alternative.)*
- [ ] Create a **Cloud Map private DNS namespace** (e.g. `yelb.local`) (requirement #12).
- [ ] Define **task definitions** and **services** for `yelb-ui`, `yelb-appserver`, `redis-server`, each registered in Cloud Map so they resolve each other by name (e.g. `redis-server.yelb.local`).
- [ ] Reuse the **RDS** Postgres from Phase 2 (or a fresh one) in the DB subnets for the database tier.
- [ ] Put the **ALB** in front of the `yelb-ui` service.
- [ ] Add **ECS service autoscaling** (target tracking on CPU or ALB request-count-per-target).
- [ ] Confirm the app works end to end through the ALB, with services finding each other via Cloud Map (no hard-coded IPs).

**DoD:** yelb is fully functional on ECS; killing a task is self-healed by the service; services communicate via Cloud Map DNS names.

**Cost note:** ECS on EC2 reuses free-tier instances; ALB + RDS remain the billable items.

---

## Phase 8 — ECS observability (Container Insights + CloudWatch + X-Ray)

**Goal:** the container equivalent of Phase 3.

**Checklist**
- [ ] Enable **Container Insights** on the ECS cluster (requirement #9) — cluster/service/task CPU, memory, network metrics.
- [ ] Configure the **`awslogs`** log driver on every container so stdout/stderr lands in CloudWatch Logs (one log group per service).
- [ ] Add the **ADOT collector as a sidecar** in each task; app containers send telemetry to `localhost`; the sidecar exports traces to **X-Ray**. Grant the **task role** `xray:PutTraceSegments`, `xray:PutTelemetryRecords`, `logs:PutLogEvents`.
- [ ] Build an ECS **dashboard** (Container Insights widgets + ALB metrics) and **alarms** (service CPU/memory, task count, 5xx).
- [ ] Confirm the **X-Ray service map** now reflects the ECS topology.

**DoD:** Container Insights shows per-service metrics; logs are flowing per service; the X-Ray service map shows traces from the ECS deployment.

**Cost note:** Container Insights adds some custom-metric cost — fine at this scale; turn it off if you pause the project for a while.

---

## Phase 9 — Load test ECS & compare autoscaling

**Goal:** repeat Phase 5 against ECS and contrast the behavior.

**Checklist**
- [ ] Point the **same Locust** test at the ECS ALB URL.
- [ ] Run the ramped load; watch **Container Insights** + the alarm: ECS **service scales out** (more tasks), then scales in.
- [ ] Compare against Phase 5: scale-out **speed**, granularity (tasks vs instances), and observability differences (Container Insights vs raw EC2 metrics).
- [ ] Write a short **comparison section** (VMs vs containers) for the project writeup.

**DoD:** Documented evidence of ECS service scaling under load, plus a written EC2-vs-ECS comparison.

**Cost note:** Short bursts only; tear down Locust afterward.

---

## Phase 10 — Promotion to Prod, documentation & teardown

**Goal:** prove the multi-env story and leave things clean.

**Checklist**
- [ ] Promote the ECS stack to the **Prod** account via the gated pipeline (tag/release + manual approval).
- [ ] Verify Prod is healthy (dashboards, a trace, a smoke test).
- [ ] Write the **runbook** in `docs/`: how to deploy, where the dashboards/traces are, how to tear down.
- [ ] Capture final **screenshots**: X-Ray service map, CloudWatch dashboards, Container Insights, autoscaling events.
- [ ] **`terraform destroy`** Dev (and Prod after the demo). Double-check the AWS console for orphaned NAT/ALB/RDS/EIP.
- [ ] Confirm the **AWS Budgets** show near-zero ongoing spend.

**DoD:** Prod deployed through the pipeline; documentation complete; both accounts torn down to (near) \$0; no orphaned billable resources.

---

## 7. Requirements coverage map

| # | Requirement | Where it's covered |
|---|---|---|
| 1 | Free 3-tier app, EC2 + containerizable, X-Ray | yelb (§2) + ADOT tracing (Phases 3, 8) |
| 2 | Terraform, low cost / free tier | §5, §6, all phases |
| 3 | GitHub CI/CD | Phase 4 |
| 4 | Multiple envs + per-env pipelines (Dev/Prod) | §4, Phase 4 |
| 5 | Logging, monitoring, tracing via CloudWatch | Phases 3, 8 |
| 6 | S3 with Gateway endpoint | Phase 1 (+ used in 5, 9) |
| 7 | EC2 Auto Scaling Groups | Phase 2, 5 |
| 8 | Locust load test + autoscaling check (EC2 & ECS) | Phases 5, 9 |
| 9 | ECS Container Insights | Phase 8 |
| 10 | VPC public / private / DB subnets | Phase 1 |
| 11 | EC2 deploy via user data | Phase 2 |
| 12 | ECS Cloud Map service mesh/discovery | Phase 7 |

---

## 8. Stretch goals (if time allows)

- Manual X-Ray spans inside the Ruby appserver around DB/Redis calls (richer traces).
- Replace the managed NAT Gateway with a self-managed NAT instance and measure the cost difference.
- Add **ElastiCache (Redis)** free-tier node instead of a self-managed Redis container.
- Blue/green or rolling deployments on ECS via CodeDeploy or ECS deployment controller.
- ECS **Service Connect** instead of plain Cloud Map service discovery, and compare.
- Synthetic canary (CloudWatch Synthetics) hitting the ALB.

---

## 9. Key references

- yelb app — https://github.com/mreferre/yelb
- ADOT collector + X-Ray — https://aws-otel.github.io/docs/getting-started/x-ray/
- ADOT on ECS (sidecar) — https://aws-otel.github.io/docs/setup/ecs/
- ADOT & X-Ray (AWS docs) — https://docs.aws.amazon.com/xray/latest/devguide/xray-services-adot.html
- ECS service discovery with Cloud Map — https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html
- X-Ray daemon on ECS — https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-ecs.html
- Locust — https://docs.locust.io/
- Terraform AWS provider — https://registry.terraform.io/providers/hashicorp/aws/latest/docs
- GitHub Actions OIDC with AWS — https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services

---

*Mentor tip: gate each phase with a short review (a 15-minute walkthrough of the DoD) before moving on. The phases build on each other, and catching a shaky VPC or IAM setup early saves hours later.*
