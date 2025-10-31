# Salesforce DevOps Release Management Automation (RMA) – V2.2 (JWT + Merge Gate + Review Approval)

This guide defines a full **Release Management Automation (RMA)** process for Salesforce using **GitHub Actions**, **JWT authentication**, and **2GP Unlocked Packages**.  
It ensures that all automated checks (**PMD**, **ESLint**, and **Validation**) **must pass**, and that a **Pull Request must be reviewed and approved**, before merging and triggering the automatic deployment to QA.

---

<details>
<summary><strong>▶ Level 1: Project & Environment Setup (JWT Auth)</strong></summary>

### 🎯 Objective
Set up Salesforce CLI, GitHub, and JWT authentication for multiple environments.  
If JWT isn’t configured yet, see [`SFDevOps-JWT-Setup.md`](SFDevOps-JWT-Setup.md).

---

### 1️⃣ Prerequisites
- [Salesforce CLI](https://developer.salesforce.com/tools/sfdxcli)  
- [Git](https://git-scm.com/downloads)  
- [Node.js 20+](https://nodejs.org/en/download/)  
- [Python 3.x](https://www.python.org/downloads/)  
- [OpenSSL](https://slproweb.com/products/Win32OpenSSL.html) for RSA keys  
- A Connected App + JWT key pair per environment (see JWT guide)

---

### 2️⃣ Authorized Environments (via JWT)
Follow [`SFDevOps-JWT-Setup.md`](SFDevOps-JWT-Setup.md) to create Connected Apps and store the following GitHub Secrets:

| Secret | Description |
|:--|:--|
| `SF_JWT_KEY` | RSA private key |
| `SF_CLIENT_ID_DEV` | Connected App Client ID (Dev) |
| `SF_CLIENT_ID_QA` | Connected App Client ID (QA) |
| `SF_CLIENT_ID_UAT` | Connected App Client ID (UAT) |
| `SF_CLIENT_ID_STAGING` | Connected App Client ID (Staging) |
| `SF_CLIENT_ID_PROD` | Connected App Client ID (Prod) |
| `SF_USERNAME_DEV` | Integration user for Dev |
| `SF_USERNAME_QA` | Integration user for QA |
| … | (repeat for other environments) |

---

### 3️⃣ Create SFDX Project & GitHub Repo
```bash
sf project generate --name MyApp
cd MyApp
git init
git add .
git commit -m "Initial MyApp project setup"
```

Create a new GitHub repo **MyApp**, copy its HTTPS URL, then:
```bash
git remote add origin https://github.com/<your-username>/MyApp.git
git branch -M main
git push -u origin main
```

---

### 4️⃣ Create Unlocked Package
```bash
sf package create --name "MyApp" --path force-app --package-type Unlocked --description "MyApp core package" --target-dev-hub devhub
```
Edit `sfdx-project.json`:
```json
"versionName": "ver 0.1",
"versionNumber": "0.1.0.NEXT"
```

Update some Salesforce Metadata on Dev Org (for example we are adding new field Nick_Name__c on Contact Object):
```bash
mkdir -p force-app/main/default/objects/Contact/fields
touch force-app/main/default/objects/Contact/fields/Nick_Name__c.field-meta.xml
```

```xml
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
  <fullName>Nick_Name__c</fullName>
  <label>Nick Name</label>
  <length>80</length>
  <type>Text</type>
</CustomField>
```

Create initial version:
```bash
sf package version create --package "MyApp" --installation-key-bypass --wait 10 --target-dev-hub devhub
```

Push updates to GitHub repo:
```bash
git add .
git commit -m "Package Creation"
git push -u origin main
```

✅ You’re ready for automation.

</details>

---

<details>
<summary><strong>▶ Level 2: Developer Automation & QA Deployment (Checks + Review Gate)</strong></summary>

### 🎯 Objective
Implement an automated **quality-gate and review approval** process:
1. **PMD**, **ESLint**, and **Validation** must pass.  
2. At least one **team member approval** required.  
3. Only then does the **Merge** button become active.  
4. On merge, QA deployment runs automatically.

---

### 1️⃣ Workflow File
Create:  
`.github/workflows/salesforce-rma.yml`
```yaml
name: Salesforce RMA Pipeline

on:
  push:
    branches: [feature/*]
  pull_request:
    branches: [main]
  workflow_dispatch:
```

---

### 2️⃣ Setup Job
```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - uses: sfdx-actions/setup-sfdx@v3
        with:
          version: latest
```

---

### 3️⃣ Validation Jobs (PMD + ESLint + Package Validation)
```yaml
  validate:
    runs-on: ubuntu-latest
    needs: setup
    steps:
      - uses: actions/checkout@v4

      - name: Run PMD (Apex)
        run: npx pmd -d force-app/main/default/classes -R pmd-ruleset.xml -f text

      - name: Run ESLint (LWC)
        run: npx eslint force-app/main/default/lwc

      - name: Validate Salesforce Package
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_DEV: ${{ secrets.SF_CLIENT_ID_DEV }}
          SF_USERNAME_DEV: ${{ secrets.SF_USERNAME_DEV }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          sf org login jwt --username $SF_USERNAME_DEV --client-id $SF_CLIENT_ID_DEV --jwt-key-file server.key --alias DevSandbox
          sf package version create --code-coverage --skip-validation false --wait 10
```

If any of these checks fail ❌,  
the workflow fails and the **Merge** button remains disabled.  
Once all pass ✅, the PR becomes mergeable.

---

### 4️⃣ Branch Protection Rules (GitHub Settings)

In your repository:  
**Settings → Branches → Add Rule → Pattern:** `main`

Enable the following options **in this order** for clarity:

1. ✅ **Require a pull request before merging**  
   - Ensures all code changes go through PR review.

2. ✅ **Require status checks to pass before merging**  
   - Blocks merging until all automated checks (PMD, ESLint, Validation) succeed.

3. ✅ **Require approvals → at least 1 approval**  
   - Requires a peer or lead to review and approve before merging.

4. ✅ **Require branches to be up to date before merging**  
   - Forces developers to rebase if main has new commits, avoiding conflicts.

5. ✅ **Include administrators** *(recommended)*  
   - Applies the same protections even to admin users.

Under **Required Status Checks**, select:
- `Run PMD (Apex)`
- `Run ESLint (LWC)`
- `Validate Salesforce Package`

🟢 **Merge button stays disabled until:**
1. All automated checks pass  
2. At least one reviewer approves  
3. The branch is up to date with main

---

### 5️⃣ Auto-Deploy to QA on Merge
```yaml
  deploy_to_qa:
    runs-on: ubuntu-latest
    needs: validate
    if: github.event.pull_request.merged == true
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to QA (on PR merge)
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_QA: ${{ secrets.SF_CLIENT_ID_QA }}
          SF_USERNAME_QA: ${{ secrets.SF_USERNAME_QA }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          sf org login jwt --username $SF_USERNAME_QA --client-id $SF_CLIENT_ID_QA --jwt-key-file server.key --alias QASandbox
          sf project deploy start --target-org QASandbox --source-dir force-app --wait 10
          sf apex run test --target-org QASandbox --wait 10

      - name: Notify Success
        if: success()
        run: echo "✅ QA deployment completed for $GITHUB_REF"

      - name: Notify Failure
        if: failure()
        run: echo "❌ QA deployment failed for $GITHUB_REF"
```

✅ Once the PR is merged after all checks and approvals,  
GitHub automatically triggers QA deployment.

</details>

---

<details>
<summary><strong>▶ Level 3: Release Manager Promotion (QA → UAT → Staging → Prod)</strong></summary>

### 🎯 Objective
Allow the Release Manager (RM) to manually promote validated builds through **UAT**, **Staging**, and **Production** using JWT authentication.

---

### 1️⃣ Manual Promotion Job
```yaml
  promote:
    runs-on: ubuntu-latest
    needs: deploy_to_qa
    if: github.event_name == 'workflow_dispatch'
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to UAT
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_UAT: ${{ secrets.SF_CLIENT_ID_UAT }}
          SF_USERNAME_UAT: ${{ secrets.SF_USERNAME_UAT }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          sf org login jwt --username $SF_USERNAME_UAT --client-id $SF_CLIENT_ID_UAT --jwt-key-file server.key --alias UATSandbox
          sf project deploy start --target-org UATSandbox --source-dir force-app --wait 10

      - name: Deploy to Staging
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_STAGING: ${{ secrets.SF_CLIENT_ID_STAGING }}
          SF_USERNAME_STAGING: ${{ secrets.SF_USERNAME_STAGING }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          sf org login jwt --username $SF_USERNAME_STAGING --client-id $SF_CLIENT_ID_STAGING --jwt-key-file server.key --alias StagingSandbox
          sf project deploy start --target-org StagingSandbox --source-dir force-app --wait 10
          sf apex run test --target-org StagingSandbox --wait 10
```

---

✅ For kicking the manual job:

```
Go to your GitHub repo.

> Click the Actions tab.
> On the left sidebar, click Salesforce RMA Pipeline (the name of your workflow).
> Click the “Run workflow” button on the right.
> Optionally pick a branch (usually main).
> Click the green Run workflow button.
```

### 2️⃣ Production Deployment (via Tag)
```yaml
  deploy_to_prod:
    runs-on: ubuntu-latest
    needs: promote
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_PROD: ${{ secrets.SF_CLIENT_ID_PROD }}
          SF_USERNAME_PROD: ${{ secrets.SF_USERNAME_PROD }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          sf org login jwt --username $SF_USERNAME_PROD --client-id $SF_CLIENT_ID_PROD --jwt-key-file server.key --alias Prod
          sf project deploy start --target-org Prod --source-dir force-app --wait 10
```

---


Push updates to GitHub workflow file to repo:
```bash
git add .github/workflows/salesforce-rma.yml
git commit -m "Add Salesforce RMA CI/CD pipeline workflow"
git push
```

### 3️⃣ To Trigger Promotion or Release
- RM runs manually from **Actions → Salesforce RMA Pipeline → Run workflow**, or  
- Tag a release:
  ```bash
  git tag v1.0.0
  git push origin v1.0.0
  ```

✅ Tag = automatic Production deployment.

</details>

---

### 🧩 End-to-End Summary

| Stage | Trigger | Validation | Approval | Target | Auth | Result |
|:--|:--|:--|:--|:--|:--|:--|
| Developer Push | `feature/*` | PMD + ESLint + Validation | — | DevSandbox | JWT | Fail → merge blocked / Pass → ready for PR |
| Pull Request Open | `PR → main` | Re-runs checks | Required ✅ | — | — | Merge locked until checks + approval |
| Merge PR | After approval + passing checks | — | — | QASandbox | JWT | Auto-deploy to QA |
| Manual Promotion | RM trigger | Smoke/Integration tests | — | UAT + Staging | JWT | Controlled rollout |
| Tag Release | `vX.Y.Z` | — | — | Prod | JWT | Auto Production deploy |

---

✅ **SFDevOps-RMA-V2.2.md (Final)**  
All PRs are gated by automated checks **and** at least one approval.  
Merges trigger automatic QA deployments, while RMs manage UAT, Staging, and Production promotions securely via JWT.
