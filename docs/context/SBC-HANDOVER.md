# SBC — handover notes

Last updated: 2026-10-14. Read this together with `CLAUDE.md` and `STATE.md`.
Owner of this fork: Krishna (GitHub `gundam-labs`).

## Where this repo came from

- Imported 2026-10-09 from the public repo `abdulfarhath7/vcfo_suite` with full
  history (395 commits) and all 5 branches. `main` = upstream `723829e`.
- The product is still branded **VCFO** everywhere (package name, UI, "Ask VCFO",
  seed emails, Terraform names `vcfo-suite`). Rebranding to SBC is not done yet.
- Upstream has no LICENSE file. Permission to reuse and extend the code is to be
  confirmed with the original author.
- `STATE.md` and `docs/context/notes/*` were written by the upstream team and
  describe **their** production deployment, not ours.

## What the app is (one paragraph)

Compliance cockpit for Indian professional-services firms: company incorporation
(mostly GCC subsidiaries of foreign parents) from client questionnaire through
MCA/SPICe+ filings, statutory registrations and the recurring compliance
calendar. Spine = 34-step checklist in four phases (`src/data/checklist.ts`).
Five roles: `super_admin`, `admin`, `manager`, `intern` (shown as Project Lead),
`client`. Workflow: lead drafts a step → manager accepts → delivered to client.

## Architecture in brief

- One Next.js 16 / React 19 / TypeScript app; UI and ~35 API routes together.
- Postgres 16 via Drizzle (35 tables), Auth.js credentials login, S3 (MinIO
  locally), Inngest jobs, docxtemplater document generation, react-pdf.
- **The seam:** only `src/db/repositories/*` may import `db`. Every repository
  function takes an `AuthContext` and scopes by role. Access control lives here,
  not in Postgres RLS ("Path A"). A missed filter = cross-tenant leak.
- Integrations: SES email (live upstream, sandbox), WhatsApp via AWS End User
  Messaging (built + tested, **off**: `WHATSAPP_ENABLED=false`, no Meta templates,
  app IAM role lacks EUM permission), Outlook via Microsoft Graph, Ask VCFO
  assistant on the Anthropic API (Haiku guard + Sonnet answer, Postgres FTS).
- Jobs (`src/jobs`): `compliance-generate` (daily 06:00), `announcement-feeds`
  (daily 06:00 IST), `whatsapp-send`, `ask-ingest`. Upstream production has **no
  Inngest runner**, so scheduled jobs do not fire there.

## Infrastructure (as found)

- Terraform, single root module in `infra/`, **one environment: prod**, AWS
  ap-south-1. App Runner (1 vCPU / 2 GB, 1–2 instances) pulling `:latest` from
  ECR with auto-deploy; RDS Postgres 16 `db.t4g.micro` single-AZ, **publicly
  accessible, 5432 open to 0.0.0.0/0** (TLS forced); S3 + KMS; Secrets Manager;
  SES; $25 budget.
- Hardcoded to the upstream AWS account `600627321277` (state bucket
  `vcfo-suite-tfstate-600627321277`, key `prod/terraform.tfstate`), their domain
  `app.sbctrack.in` and their Entra app. **Do not run `terraform apply` as-is.**
- No CI/CD (no `.github/workflows`). Deploys are manual: build → push `:latest`
  → App Runner redeploys; migrations run by hand from a laptop.

## Decisions so far

- Stay on **AWS** and enhance what exists (no move to Azure).

## Planned next steps (in order)

1. Make `gundam-labs/SBC` private (currently public); confirm reuse rights.
2. Retarget Terraform to our own AWS account; parameterise for `dev` and `prod`
   (separate state keys + tfvars).
3. GitHub Actions: typecheck + test → build → push git-SHA tag → migrate →
   deploy dev → gated promote to prod; OIDC to AWS, no long-lived keys.
4. Private RDS + App Runner VPC connector ("option A").
5. Production job runner (Inngest Cloud, or EventBridge Scheduler); SES
   production access + DKIM.
6. WhatsApp: Meta templates + phone number in EUM, IAM permission,
   `WHATSAPP_ENABLED=true`.
7. Rebrand VCFO → SBC.

Also outstanding: cross-tenant repository tests; purge demo seed users from any
deployed database (passwords are in the README).

## Architecture document

Full write-up with diagram (Claude doc, owner can share):
https://claude.ai/code/artifact/f0a9bb92-da87-4ffa-bbc9-128d346190b1
