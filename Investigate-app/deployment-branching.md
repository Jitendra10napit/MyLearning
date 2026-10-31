I'll check the Drive for anything on Investigate's build and deployment process.

I couldn't find a CI/CD or pipeline document in your Drive. The architecture notes do cover three relevant things, though: the **branching and release strategy**, a custom **Deployment Application**, and how config and secrets are handled. Here's what those show, and where you'll need to fill in from your own memory of the actual pipelines.

## The big picture

```
Developer → feature/bug branch → PR (2 reviewers + CI checks) → main → auto-deploy to DEV
                                                                  │
                                       release time: cut support/INV_SP270 (v2.7.0)
                                                                  │
                         develop/INV_SP271 (minor) or hotfix/INV_SP271+HF (critical)
                                                                  │
                                    tag v2.7.1 → Deployment Application → QA / UAT / PROD
```

## 1. Branching strategy (from the Investigator Portal branching guide)

| Branch | Example | Purpose |
|---|---|---|
| `main` | — | Ongoing development. Must always build, and is **deployed automatically to dev** |
| `feature/*`, `bug/*` | `feature/12345-new-search-filter` | Short-lived, named after the work item, merged by PR |
| `support/*` | `support/INV_SP270` | **One per major release. Production is deployed from here**, and several are kept alive at once |
| `develop/*` | `develop/INV_SP271` | Collects fixes for a minor release, tested together, then merged into the support branch |
| `hotfix/*` | `hotfix/INV_SP271+HF` | Critical production fix, lives hours to 1–2 days |
| `code/<initials>/*` | `code/cp/redaction-perf-poc` | POCs and spikes, never merged directly |

**PR rules for `main` and `support/*`:** at least 2 reviewers, passing CI/CD checks, updated tests and docs, a linked work item, and before/after screenshots.

**Versioning:** semantic versions (v2.7.0 → v2.7.1), with hotfixes tagged `+HF` (v2.6.1+HF).

**Why support branches:** each agency upgrades on its own schedule. Pune might be on 2.7 while Mumbai is still on 2.6, so the team has to patch both versions in parallel. That's the main reason this isn't plain trunk-based development.

## 2. Build (CI)

The docs don't show the pipeline YAML. Your resume lists Azure DevOps, and the work-item-linked branches and PRs fit that. The repos use **central package management** (`Directory.Packages.props`), so every project in a repo builds against the same NuGet versions.

A typical build pipeline for one service looks like this (illustrative, not NICE's actual file):

```yaml
trigger:
  branches: { include: [ main, support/*, develop/*, hotfix/* ] }
pr:
  branches: { include: [ main, support/* ] }

stages:
- stage: Build
  jobs:
  - job: BuildTest
    pool: { vmImage: windows-latest }
    steps:
    - task: UseDotNet@2
      inputs: { version: 10.x }
    - script: dotnet restore
    - script: dotnet build -c Release --no-restore
    - script: dotnet test -c Release --no-build --collect:"XPlat Code Coverage"
    - script: dotnet publish src/CaseAndEvidence.Api -c Release -o $(Build.ArtifactStagingDirectory)/api
    - script: dotnet ef migrations bundle -p src/CaseAndEvidence.Migrations -o $(Build.ArtifactStagingDirectory)/efbundle
    - publish: $(Build.ArtifactStagingDirectory)
      artifact: caseandevidence-$(Build.BuildNumber)
```

Builds of PR branches only verify the change (build plus tests). Builds of `main`, `support/*` and `hotfix/*` also publish **versioned artifacts** that can be deployed.

## 3. Deployment: the custom Deployment Application

This part *is* in the docs. NICE built its own **Deployment Application**, a web app that runs on **Hangfire** with its own database (`DeploymentApplicationDb`), and it deploys the platform to Azure environments.

**Why build a custom deployer instead of only using pipelines?** The platform has 20+ deployable parts (websites, APIs, WebJobs, Functions, Durable workflows), multiple databases, and **many agency environments, each on its own version with its own configuration**. A standard pipeline handles one service well. Upgrading one agency's whole environment, in the right order, with retries and history, is a different job.

Hangfire fits that job: each deployment step becomes a background job, steps can be chained, failed steps retry, and its dashboard shows exactly where a deployment stopped. Here's an illustrative sketch:

```csharp
// "Upgrade pune-police PROD to v2.7.1"
var backup  = BackgroundJob.Enqueue<DeploySteps>(s => s.BackupDatabases("pune-police", "prod"));
var migrate = BackgroundJob.ContinueJobWith<DeploySteps>(backup,  s => s.RunMigrations("pune-police", "v2.7.1"));
var infra   = BackgroundJob.ContinueJobWith<DeploySteps>(migrate, s => s.ApplyConfig("pune-police"));   // Key Vault, customer config
var core    = BackgroundJob.ContinueJobWith<DeploySteps>(infra,   s => s.DeployServices("v2.7.1", Tier.Platform)); // TokenService, UserService, Adjudication
var domain  = BackgroundJob.ContinueJobWith<DeploySteps>(core,    s => s.DeployServices("v2.7.1", Tier.Domain));   // CaseAndEvidence, QueryService, Functions
var portals = BackgroundJob.ContinueJobWith<DeploySteps>(domain,  s => s.DeployServices("v2.7.1", Tier.Portals)); // Investigator, Admin, Public, Business
BackgroundJob.ContinueJobWith<DeploySteps>(portals, s => s.SmokeTest("pune-police", "prod"));
```

The order matters. Identity and authorization services come first, then domain services, and the portals last, so a new UI never calls an API that hasn't been deployed yet.

The repo also has a **UI test project** (`InvDeploy.WebUI.Tests`, with environment test data), so the deployment tool itself is tested.

## 4. Configuration and secrets

- **Key Vault** holds secrets. The portal and ExternalApi depend on it directly, so connection strings aren't kept in the app's own settings.
- **Per-customer configuration lives in separate repos** (`Investigate.CustomerConfigurations.*`). The same build artifact is deployed to every agency, and only the configuration changes.
- **Database migrations ship as separate projects** (for example `MediaService.Database.Migrations`) and run as their own deployment step, not when the app starts up.
- Every repo has a **Backstage catalog file**, which records what each service depends on.

## 5. End-to-end example

A Pune investigator reports that search crashes on a special character, in version 2.7.0.

1. A developer branches `bug/45120-fix-search-crash` from `support/INV_SP270` and fixes it in QueryService.
2. They raise a PR to `develop/INV_SP271`. CI builds and tests it, and 2 reviewers approve with before/after screenshots attached.
3. QA tests `develop/INV_SP271` as a whole, it's merged to `support/INV_SP270`, and it's tagged **v2.7.1**.
4. The fix is also merged to `main`, so 2.8 doesn't bring the bug back.
5. In the Deployment Application, ops picks "pune-police / PROD / v2.7.1". The Hangfire chain runs: backup, migrations, config, then services tier by tier, then the smoke test.
6. Mumbai, still on 2.6, isn't affected. If Mumbai needs the fix, it goes into `support/INV_SP260` as v2.6.2.

## How to say it in the interview

> "We used a release-branch model because each agency upgrades on its own schedule. Main is continuous development and auto-deploys to dev. Each major release gets a long-lived support branch, minor releases are prepared in develop branches, and critical fixes go through short hotfix branches, all with semantic version tags. Every PR needs two reviewers and passing CI. Deployment is done by an in-house Deployment Application built on Hangfire: it upgrades a customer environment as a chain of jobs (backup, database migrations, configuration from Key Vault and customer config repos, then services in dependency order, then smoke tests), with retries and a full history. The same artifact goes to every agency and only the configuration differs."

**Likely follow-ups to prepare from your own experience** (the docs don't cover these):
- *Rollback:* do you redeploy the previous tag, or use App Service deployment slots? And how do you handle database migrations that can't be undone?
- *Zero downtime:* slot swaps, and migrations that only add columns or tables first and remove old ones in a later release.
- *Hosting:* WebJobs only run on App Service, so at least some services are on App Service. Check whether any run in containers.
- *Infrastructure as code:* is Azure infrastructure created with ARM/Bicep or Terraform, or by the Deployment Application?

Sources:
- [branching-strategy-guide.md](https://drive.google.com/file/d/13w69T7dT6mJ4GDH65pcGqgQsXoktfWfC/view?usp=drivesdk)
- [00-overview.md](https://drive.google.com/file/d/1LvgBzLkGIHoc1HNzq2Fxi5-pZeGmprGV/view?usp=drivesdk)
- [02-component-dependencies.md](https://drive.google.com/file/d/1hv-RHZpG49EVcQdaDa5XqrnwahQ_zAQj/view?usp=drivesdk)
- [05-tech-and-patterns.md](https://drive.google.com/file/d/1ijIil2XvoT9PXoy3VAkeLk-wMLbFjeCP/view?usp=drivesdk)


That settles the earlier question: Investigate uses a **database per agency**. Combined with what the docs show, each agency gets its own copy of every database:

```
pune-police                          mumbai-police
 ├─ InvestigateDb                     ├─ InvestigateDb
 ├─ CaseAndEvidenceDb                 ├─ CaseAndEvidenceDb
 ├─ AdminDb                           ├─ AdminDb
 ├─ TaskDb                            ├─ TaskDb
 ├─ ES: investigate-pune-police-*     ├─ ES: investigate-mumbai-police-*
 └─ Blob: pune-police                 └─ Blob: mumbai-police
```

The data is split two ways: **by bounded context** (cases, admin, tasks each have their own database) and **by agency** (each agency has its own set of them).

## Why it fits this product

- **Isolation:** criminal evidence from one agency physically can't appear in another agency's query. You don't depend on every developer remembering a `TenantId` filter.
- **Independent versions:** this is what makes the support-branch model work. Pune can be on 2.7.1 while Mumbai stays on 2.6.2, because each agency's schema gets migrated separately.
- **Compliance:** separate backup and restore per agency, data kept in the agency's region, and per-agency retention and deletion.
- **No noisy neighbour:** a heavy agency only slows down its own database.

## How the app connects to the right database

The connection string comes from the agency's configuration (in Key Vault), not from a `TenantId` column:

```csharp
services.AddDbContext<CaseAndEvidenceDbContext>((sp, opt) =>
{
    var agency = sp.GetRequiredService<IAgencyContext>().AgencyId;              // from the token claim
    var conn   = sp.GetRequiredService<IAgencyConfig>().GetConnection(agency, "CaseAndEvidenceDb"); // Key Vault
    opt.UseSqlServer(conn);
});
```

The data-access code stays simple: no global query filters and no `TenantId` columns.

## How it connects to the deployment story

This also explains why the Deployment Application deploys **per agency**. "Upgrade pune-police to v2.7.1" means running the migrations against *Pune's* databases only. Mumbai's schema stays untouched until Mumbai is upgraded.

**One thing to confirm before an interview:** if agencies can be on different versions, are the **application services also deployed separately per agency** (each agency has its own App Services and Functions, sometimes called a deployment stamp), or does one shared deployment serve all agencies with only the databases split? Different versions at the same time strongly suggests separate deployments per agency, but you'll know for sure. Interviewers often ask this follow-up.

## The trade-offs (be ready to name them)

| Cost | How it's handled |
|---|---|
| More databases to pay for | Azure SQL Elastic Pools share compute across agency databases |
| Migrations across N agencies | Run per agency by the Deployment Application, with each agency on its own schedule |
| Reporting across agencies | A separate pipeline or warehouse, never direct queries across databases |
| Onboarding a new agency | Create the databases, index and container, then add the config entry |

## How to say it in the interview

> "Every agency has its own set of databases, one per bounded context: InvestigateDb, CaseAndEvidenceDb, AdminDb, TaskDb. It also has its own Elasticsearch indexes and blob container. The agency comes from the token claim, and its connection strings come from Key Vault. We chose physical isolation because this is law-enforcement evidence: it gives us audit-proof separation, per-agency backup and residency, and it lets each agency upgrade on its own schedule, because the Deployment Application migrates one agency's databases at a time. We keep costs down with elastic pools."
