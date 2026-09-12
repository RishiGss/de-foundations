# IAM Design

Living record of every identity in this programme — who/what it is, what it can touch, why it exists, and where it's used. 
Update this in the same PR that changes any IAM-relevant resource; don't let it drift from what Terraform actually applies.

**Naming convention:** `<function>-<qualifier>-sa`, lowercase, hyphens, ≤30 chars. Never named after the programme/learning plan itself — a name should be readable cold, without knowing this document exists.

---

## Principle

Provisioning identities, orchestration identities, and runtime identities are three different blast radii and must never collapse into one SA.  
An identity that creates infrastructure should not be the same identity that runs unattended workloads against production data.  
This is the single most common finding in real IAM security reviews, and it's the standard this repo holds itself to from Day 1.

---

## Active identities

### `dev-tf-provisioner-sa`

| **Property** | **Info** |
|---|---|
| **Project** | `learning-dataeng-dev` |
| **Created** | W1, Sat 05 Sep 2026 |
| **Purpose** | Terraform's identity for provisioning infra in dev — buckets, datasets, and (later) networking, Pub/Sub |
| **Roles held** | `roles/storage.admin`, `roles/bigquery.admin` |
| **Used by** | Terraform only, via `impersonate_service_account` in `environments/dev/providers.tf` |
| **Auth method** | Keyless impersonation. No JSON key exists. |
| **Who can impersonate it** | `gssrishi@gmail.com` — holds `roles/iam.serviceAccountTokenCreator` scoped to this SA only, not project-wide |
| **Deletion protection** | `deletion_policy = "PREVENT"` (GCP-side) |
| **Bootstrap note** | Created via `bootstrap/` using local state + the operator's own ADC — this is the one config in the repo that doesn't impersonate the SA, because the SA doesn't exist yet at that point |
| **Never gets** | Runtime/data-plane roles (Dataproc worker, dataEditor). If it starts needing those, that's a sign work is being done by the wrong identity. |

### `dev-tf-state-bucket` (not a service account — the state backend)

| **Property** | **Info** |
|---|---|
| **Name** | `learning-dataeng-dev-tfstate-bucket` |
| **Created** | W1, via `bootstrap/` |
| **Bucket Metadata Properties** | Versioning ON, `uniform_bucket_level_access = true`, `public_access_prevention = "enforced"` |
| **Protections** | `deletion_policy = "PREVENT"`, `lifecycle { prevent_destroy = true }` (both layers deliberately, not redundant — one is GCP-side, one is Terraform-side) |
| **Note** | This is the one bucket in the whole repo that should never lose its destroy protection, for its entire lifetime |

---

## Planned identities — full roadmap, decided ahead of need

| SA | When | Roles | Used by | Protection |
|---|---|---|---|---|
| `dev-tf-provisioner-sa` | ✅ W1 | `storage.admin`, `bigquery.admin` → +networking (W3) → +`pubsub.admin` (W9) | Terraform, dev only | `PREVENT` |
| `dataproc-runtime-sa` | W2 | `dataproc.worker`, `storage.objectAdmin` (bucket-scoped), `bigquery.dataEditor`, `bigquery.jobUser` | Airflow → Dataproc Serverless batches | None — least-privilege, unattended, no admin roles by design |
| `airflow-orchestrator-sa` | W2 | `dataproc.editor`, `iam.serviceAccountUser` on `dataproc-runtime-sa` | Airflow scheduler, to submit batches | None |
| GitHub Actions WIF identity | W4 | Narrow, repo-scoped | CI pipeline only | N/A — WIF, not a standing SA |
| `prd-tf-provisioner-sa` | W17 | Mirrors `dev-tf-provisioner-sa`'s role progression | Terraform, prod only | `PREVENT` — non-negotiable at this stage |

**Rule:** `dev-tf-provisioner-sa` stops growing at provisioning permissions. Any new runtime need gets a new SA, not a new role on this one.

---

## Data-plane resources (not identities, but scoped here for context)

### Dev Project - `learning-dataeng-dev`

#### GCS Buckets
Location: asia-south1

**Datalake Buckets**  
Protection: None yet — deliberately, to allow the W1 destroy/apply drill. Add `PREVENT` once real, non-trivially-re-ingestible data lands (~W5–W6).  
Bucket Names:
- `learning-dataeng-dev-raw`
- `learning-dataeng-dev-curated`

**Airflow Logs Bucket**  
Protection: None planned — disposable operational logs  
Bucket Name: `learning-dataeng-dev-airflow-logs`

#### BQ Datasets
Location: asia-south1  
Protection: None — these are derived/rebuildable via Spark/dbt, protection would block routine rebuild drills (dbt schema iteration, SCD-2 redos)  
BQ Datasets based on Medallion Architecture:
- `bronze`
- `silver`
- `gold`

---

## Bootstrap problem — documented once, referenced everywhere

Any config creating an identity or backend that a *later* config depends on cannot use that identity/backend for its own creation. Solved via `bootstrap/`, which:
- runs on local state
- authenticates as the human operator's own ADC, never an SA
- creates exactly the identity + bucket that everything else then depends on

This pattern repeats identically for prod bootstrap in W17.

---

## Revision log

| Date | Change |
|---|---|
| 2026-09-05 | `dev-tf-provisioner-sa` created, bootstrap established |
| 2026-09-12 | GCS/BQ dev footprint (`raw`, `curated`, `airflow-logs`, `bronze`, `silver`, `gold`) provisioned via `dev-tf-provisioner-sa` |