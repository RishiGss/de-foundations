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
| **Roles held** | `roles/storage.admin` (provision/manage GCS buckets), `roles/bigquery.admin` (provision/manage BQ datasets), `roles/iam.serviceAccountAdmin` (create SAs, set IAM policy on individual SAs), `roles/resourcemanager.projectIamAdmin` (set project-level IAM policy) |
| **Why IAM-admin roles** | Provisioning includes *identity* provisioning. `storage.admin` + `bigquery.admin` alone cannot create service accounts or modify project IAM policy — discovered W2 when `dataproc-runtime-sa` creation failed with `iam.serviceAccounts.create` denied. `serviceAccountAdmin` covers creating SAs and setting IAM policy *on* an SA (the impersonation grants); `projectIamAdmin` covers project-level role bindings. |
| **Known escalation risk** | `projectIamAdmin` means this SA can grant itself any role on the project. Accepted deliberately: this identity is only ever assumed interactively, by the operator, during a deliberate `terraform apply` — never wired into an unattended workload. That constraint is what makes the risk tolerable, and it is the reason runtime SAs get neither of these roles. |
| **Used by** | Terraform only, via `impersonate_service_account` in `environments/dev/providers.tf` |
| **Auth method** | Keyless impersonation. No JSON key exists. |
| **Who can impersonate it** | `gssrishi@gmail.com` — holds `roles/iam.serviceAccountTokenCreator` scoped to this SA only, not project-wide |
| **Deletion protection** | `deletion_policy = "PREVENT"` (GCP-side) |
| **Bootstrap note** | Created via `bootstrap/` using local state + the operator's own ADC — this is the one config in the repo that doesn't impersonate the SA, because the SA doesn't exist yet at that point |
| **Never gets** | Runtime/data-plane roles (Dataproc worker, dataEditor, bucket-scoped object roles) — those belong to `dataproc-runtime-sa`. If this SA starts needing them, that's a sign work is being done by the wrong identity. |

### `dataproc-runtime-sa`

| **Property** | **Info** |
|---|---|
| **Project** | `learning-dataeng-dev` |
| **Created** | W2, Sat 12 Sep 2026 — via `environments/dev`, not `bootstrap/` |
| **Purpose** | Least-privilege runtime identity for Dataproc Serverless batches submitted by Airflow |
| **Project roles** | `roles/dataproc.worker`, `roles/bigquery.dataEditor`, `roles/bigquery.jobUser` |
| **Bucket-scoped roles** | `roles/storage.objectAdmin` on `learning-dataeng-dev-raw`, `-curated`, `-airflow-logs` — granted per-bucket via `google_storage_bucket_iam_member`, deliberately not project-wide `storage.admin` |
| **Who can impersonate it** | `gssrishi@gmail.com`, `roles/iam.serviceAccountTokenCreator` scoped to this SA only |
| **Deletion protection** | `deletion_policy = "PREVENT"` |
| **Verified capabilities** | Impersonation confirmed (token mint); write access confirmed via object upload to `raw` bucket |
| **Known limitation — not a scoping failure** | `roles/dataproc.worker` bundles broad, unscoped storage permissions (`storage.objects.get/list/create/delete`, `storage.buckets.get`) as an inherent property of the predefined role — confirmed via `gcloud iam roles describe roles/dataproc.worker`. Since this role must be granted at project level for Dataproc Serverless to function, this SA can read/list objects in *any* bucket in the project, including the Terraform state bucket, regardless of the bucket-scoped grants below. Bucket-scoped grants still correctly limit *write* intent to the three data buckets. Accepted trade-off — a custom role would close this gap but was judged disproportionate overhead for a solo learning environment. |
| **Why it could access tfstate — investigation trail** | Negative test (`gcloud storage ls`/`cat` on the tfstate bucket, impersonating this SA) unexpectedly succeeded. Ruled out in order: (1) legacy bucket ACL inheritance — bucket had `uniform_bucket_level_access = true`, removing legacy Reader bindings made no difference; (2) unintended project-level role — confirmed via `gcloud projects get-iam-policy`, this SA holds only its three intended roles. Root cause confirmed via `gcloud iam roles describe roles/dataproc.worker`. |
| **Never gets** | Any `*.admin` role, project-wide storage access, or IAM-admin roles. It runs unattended on a schedule — a bug in a DAG must not be able to delete a bucket or alter IAM. |

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
| `dev-tf-provisioner-sa` | ✅ W1 | `storage.admin`, `bigquery.admin`, `iam.serviceAccountAdmin`, `resourcemanager.projectIamAdmin` → +networking (W3) → +`pubsub.admin` (W9) | Terraform, dev only | `PREVENT` |
| `dataproc-runtime-sa` | ✅ W2 | `dataproc.worker`, `bigquery.dataEditor`, `bigquery.jobUser`, `storage.objectAdmin` (bucket-scoped) | Airflow → Dataproc Serverless batches | `PREVENT` |
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
| 2026-09-12 | `dataproc-runtime-sa` created — first runtime identity. Project roles + bucket-scoped `objectAdmin` on the three data buckets. |
| 2026-09-12 | `dev-tf-provisioner-sa` granted `iam.serviceAccountAdmin` + `resourcemanager.projectIamAdmin`. Cause: creating a second SA through `environments/dev` failed — the provisioner SA could provision storage and BQ but not identities or IAM bindings. Escalation risk accepted; see the SA's own section. |
| 2026-09-12 | Negative test on `dataproc-runtime-sa` found it can read the tfstate bucket via `roles/dataproc.worker`'s bundled storage permissions — documented as a known, accepted limitation, not a misconfiguration. |
