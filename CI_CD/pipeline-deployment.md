Here's a short first-person version you can speak in about 60–90 seconds. Adjust any detail that doesn't match what you actually did, because interviewers dig into specifics.

## Spoken answer

"In my projects we followed **build once, deploy many**.

On every pull request, the **CI** pipeline in Azure DevOps restores, builds, runs unit tests and code coverage, does a SonarQube scan and dependency scanning, then builds the Docker image and pushes it to **ACR, tagged with the commit SHA**. PRs can't merge unless these quality gates pass.

For **CD**, the same image moves through **Dev → QA → Staging → Prod**. Only configuration changes per environment, through Key Vault and App Configuration. Dev and QA deploy automatically, and Prod needs approval.

Each microservice has **its own pipeline** with path-based triggers, built from **shared YAML templates**, so every team gets the same security and quality steps.

Infrastructure is created with **Bicep or Terraform** through the pipeline, never manually in the portal.

For **zero downtime** we used **deployment slots** for App Service and Azure Functions, and **canary or revision traffic splitting** for containers. Database changes are **backward-compatible** (expand and contract), so rollback is just a slot swap or redeploying the previous image tag.

After deployment, **Application Insights** health checks act as a gate, and we tracked deployment frequency and failure rate to keep improving."

## One-line memory hook: "B-T-S-P-I-Z-M"

| Letter | Stands for |
|---|---|
| **B** | Build once, deploy many (image tagged with commit SHA) |
| **T** | Tests and quality gates on every PR |
| **S** | Security scans and secrets in Key Vault |
| **P** | Pipeline per microservice, with shared templates |
| **I** | Infrastructure as Code (Bicep/Terraform) |
| **Z** | Zero downtime: slots, canary, backward-compatible DB changes |
| **M** | Monitor with App Insights, rollback, DORA metrics |

## Tie it to your NICE work

"For example, the **Azure Functions compute service** and the **Durable Functions buffer service** each had separate pipelines. We deployed Functions to a **staging slot**, ran smoke tests, then **swapped** to production. Rollback was simply swapping back."



**Blue-Green Deployment**

You keep two identical environments:
- **Blue** is the live version that users are on now.
- **Green** is where you deploy the new version.

You test Green. When it's fine, you **switch all traffic from Blue to Green at once**. If something goes wrong, you switch back to Blue immediately.

- **Pro:** instant switch and instant rollback.
- **Con:** you need double the infrastructure (cost).
- **Azure example:** App Service or Azure Functions **deployment slots**. Deploy to the staging slot, then **swap**.

**Canary Deployment**

You release the new version to a **small group of users first**, for example 5%. You watch errors and performance. If everything looks good, you increase it step by step: 5% → 25% → 50% → 100%. If problems appear, you send traffic back to the old version.

- **Pro:** low risk, because only a few users are affected if there's a bug.
- **Con:** slower rollout, needs good monitoring, and the old and new versions run together.
- **Azure example:** Container Apps **revision traffic splitting**, AKS with Istio, or Front Door weights.

**One-line difference**

- **Blue-Green:** switch **everyone at once** between two environments.
- **Canary:** move users **gradually**, a small percentage first.

**Easy way to remember:** *Canary* comes from the "canary in a coal mine": test on a few first, and if they're safe, everyone goes.
