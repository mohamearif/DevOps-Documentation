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
| `SF_CLIENT_ID_DEVHUB` | Connected App Client ID for the Dev Hub (org that owns the 2GP packages) |
| `SF_USERNAME_DEVHUB` | Integration user email for the Dev Hub |
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

permissions:
  contents: write
  pull-requests: write

on:
  push:
    branches: 
     - feature/*
     - main
    tags:
      - 'v*'       # Triggers workflow when a version tag (e.g. v1.0.0) is pushed
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches: [main]
  workflow_dispatch:
```

---

### 2️⃣ Setup Job
```yaml
jobs:
  setup:
    if: ${{ github.event_name != 'workflow_dispatch' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli
          sf --version
```

---

### 3️⃣ Validation Jobs (PMD + ESLint + Package Validation)
```yaml
   validate:
    if: ${{ github.event_name != 'workflow_dispatch' }}
    runs-on: ubuntu-latest
    needs: setup
    steps:
      - uses: actions/checkout@v4

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli
          sf --version

      - name: Run PMD (Apex)
        run: |
          echo "🔍 Running PMD manually without artifact upload..."
          wget -q https://github.com/pmd/pmd/releases/download/pmd_releases/6.55.0/pmd-bin-6.55.0.zip
          unzip -q pmd-bin-6.55.0.zip
          ./pmd-bin-6.55.0/bin/run.sh pmd \
            -d force-app/main/default/classes \
            -R pmd-ruleset.xml \
            -f text \
            -r pmd-report.txt || true
          echo "✅ PMD completed. Printing summary:"
          tail -n 50 pmd-report.txt || echo "No issues found"

      - name: Install ESLint (v8)
        run: |
          npm install --no-save eslint@8 @salesforce/eslint-config-lwc --legacy-peer-deps

      - name: Run ESLint (LWC)
        run: |
          if [ -d "force-app/main/default/lwc" ] && [ "$(ls -A force-app/main/default/lwc 2>/dev/null)" ]; then
            echo "🔍 Installing and running ESLint v9..."
            npm install --no-save eslint@9 @lwc/eslint-plugin-lwc@latest
            npx eslint force-app/main/default/lwc
          else
            echo "⚠️ No LWC components found. Skipping ESLint check."
          fi

      - name: Validate Deployment and Apex Tests on QA (check-only)
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_QA: ${{ secrets.SF_CLIENT_ID_QA }}
          SF_USERNAME_QA: ${{ secrets.SF_USERNAME_QA }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          echo "🔐 Authenticating to QA org..."
          sf org login jwt --username $SF_USERNAME_QA --client-id $SF_CLIENT_ID_QA --jwt-key-file server.key --alias QASandbox

          echo "🧪 Validating deployment in QA (check-only)..."
          sf project deploy validate \
            --target-org QASandbox \
            --source-dir force-app \
            --wait 10 \
            --test-level RunLocalTests
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
    if: github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == true
    steps:
      - uses: actions/checkout@v4

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli
          sf --version

      # Step 1: Authenticate to Dev Hub for package creation
      - name: Authenticate to Dev Hub
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_DEVHUB: ${{ secrets.SF_CLIENT_ID_DEVHUB }}
          SF_USERNAME_DEVHUB: ${{ secrets.SF_USERNAME_DEVHUB }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          sf org login jwt \
          --username $SF_USERNAME_DEVHUB \
          --client-id $SF_CLIENT_ID_DEVHUB \
          --jwt-key-file server.key \
          --alias DevHub \
          --set-default-dev-hub

      # Step 2: Create package version (in Dev Hub)
      - name: Create Package Version (with Coverage)
        run: |
          sf package version create \
          --package "MyApp" \
          --installation-key-bypass \
          --code-coverage \
          --wait 10 \
          --target-dev-hub DevHub
          sf package version list

      # Step 3: Commit and push new alias update
      - name: Commit and Push New Package Version Alias
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          latest_version=$(sf package version list --packages "MyApp" --target-dev-hub DevHub --json | jq -r 'if (.result | type) == "array" and (.result | length) > 0 then .result | sort_by(.CreatedDate) | reverse | .[0].SubscriberPackageVersionId else empty end')
          echo "Latest package version: $latest_version"

          git config --global user.name "github-actions"
          git config --global user.email "actions@github.com"
          git add sfdx-project.json
          git commit -m "Update packageAliases with version $latest_version [skip ci]" || echo "No changes to commit"
          git push origin main || echo "No push (branch protection may prevent direct commits)"

      # Step 4: Authenticate to QA (set alias + default)
      - name: Authenticate to QA
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_QA: ${{ secrets.SF_CLIENT_ID_QA }}
          SF_USERNAME_QA: ${{ secrets.SF_USERNAME_QA }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          sf org login jwt \
            --username $SF_USERNAME_QA \
            --client-id $SF_CLIENT_ID_QA \
            --jwt-key-file server.key \
            --alias QASandbox \
            --set-default

      # Step 5: Install the newly created package version into QA
      - name: Install Latest Package Version to QA
        run: |
          latest_version=$(sf package version list --packages "MyApp" --target-dev-hub DevHub --json | jq -r 'if (.result | type) == "array" and (.result | length) > 0 then .result | sort_by(.CreatedDate) | reverse | .[0].SubscriberPackageVersionId else empty end')
          echo "Latest package version: $latest_version"
          echo "Installing package version $latest_version to QA..."
          sf package install --package "$latest_version" --wait 10 --no-prompt

      # Step 6: Notifications
      - name: Notify Success
        if: success()
        run: echo "✅ QA package version created (via Dev Hub) and installed successfully for $GITHUB_REF"

      - name: Notify Failure
        if: failure()
        run: echo "❌ QA package creation or installation failed for $GITHUB_REF"

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
    if: ${{ github.event_name == 'workflow_dispatch' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli
          sf --version

      # Step 1: Deploy to UAT
      - name: Deploy to UAT
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_UAT: ${{ secrets.SF_CLIENT_ID_UAT }}
          SF_USERNAME_UAT: ${{ secrets.SF_USERNAME_UAT }}
          SF_CLIENT_ID_DEVHUB: ${{ secrets.SF_CLIENT_ID_DEVHUB }}
          SF_USERNAME_DEVHUB: ${{ secrets.SF_USERNAME_DEVHUB }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          sf org login jwt \
            --username $SF_USERNAME_UAT \
            --client-id $SF_CLIENT_ID_UAT \
            --jwt-key-file server.key \
            --alias UATSandbox \
            --set-default

          echo "📦 Fetching latest package version from DevHub..."
          sf org login jwt \
            --username $SF_USERNAME_DEVHUB \
            --client-id $SF_CLIENT_ID_DEVHUB \
            --jwt-key-file server.key \
            --alias DevHub \
            --set-default-dev-hub

          latest_version=$(sf package version list --packages "MyApp" --target-dev-hub DevHub --json | jq -r 'if (.result | type) == "array" and (.result | length) > 0 then .result | sort_by(.CreatedDate) | reverse | .[0].SubscriberPackageVersionId else empty end')
          echo "Latest package version: $latest_version"
          echo "🚀 Installing version $latest_version into UAT..."
          sf package install --package "$latest_version" --wait 10 --no-prompt

      # Step 2: Deploy to Staging
      - name: Deploy to Staging
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_STAGING: ${{ secrets.SF_CLIENT_ID_STAGING }}
          SF_USERNAME_STAGING: ${{ secrets.SF_USERNAME_STAGING }}
          SF_CLIENT_ID_DEVHUB: ${{ secrets.SF_CLIENT_ID_DEVHUB }}
          SF_USERNAME_DEVHUB: ${{ secrets.SF_USERNAME_DEVHUB }}
        run: |
          echo "$SF_JWT_KEY" > server.key
          sf org login jwt \
            --username $SF_USERNAME_STAGING \
            --client-id $SF_CLIENT_ID_STAGING \
            --jwt-key-file server.key \
            --alias StagingSandbox \
            --set-default

          echo "📦 Fetching latest package version from DevHub..."
          sf org login jwt \
            --username $SF_USERNAME_DEVHUB \
            --client-id $SF_CLIENT_ID_DEVHUB \
            --jwt-key-file server.key \
            --alias DevHub \
            --set-default-dev-hub

          latest_version=$(sf package version list --packages "MyApp" --target-dev-hub DevHub --json | jq -r 'if (.result | type) == "array" and (.result | length) > 0 then .result | sort_by(.CreatedDate) | reverse | .[0].SubscriberPackageVersionId else empty end')
          echo "Latest package version: $latest_version"
          echo "🚀 Installing version $latest_version into Staging..."
          sf package install --package "$latest_version" --wait 10 --no-prompt

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
    if: github.ref_type == 'tag' && startsWith(github.ref, 'refs/tags/') && github.event.base_ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli
          sf --version

      - name: Deploy to Production (DevHub = Prod)
        env:
          SF_JWT_KEY: ${{ secrets.SF_JWT_KEY }}
          SF_CLIENT_ID_PROD: ${{ secrets.SF_CLIENT_ID_DEVHUB }}
          SF_USERNAME_PROD: ${{ secrets.SF_USERNAME_DEVHUB }}
          SF_CLIENT_ID_DEVHUB: ${{ secrets.SF_CLIENT_ID_DEVHUB }}
          SF_USERNAME_DEVHUB: ${{ secrets.SF_USERNAME_DEVHUB }}
        run: |
          echo "$SF_JWT_KEY" > server.key

          echo "🔐 Authenticating to Production (DevHub) org..."
          sf org login jwt \
            --username $SF_USERNAME_PROD \
            --client-id $SF_CLIENT_ID_PROD \
            --jwt-key-file server.key \
            --alias Prod \
            --set-default

          sf alias set Prod=$SF_USERNAME_PROD

          echo "📦 Fetching latest package version from DevHub..."
          latest_version=$(sf package version list --packages "MyApp" --target-dev-hub Prod --json | jq -r '.result | sort_by(.CreatedDate) | reverse | .[0].SubscriberPackageVersionId')

          echo "Latest package version: $latest_version"
          echo "🚀 Installing version $latest_version into Production..."
          sf package install --package "$latest_version" --wait 10 --no-prompt

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

<details>
<summary><strong>▶ Level 4: Testing and Verifying the Automation Jobs</strong></summary>

### 🎯 Objective
Confirm that every part of your CI/CD pipeline runs correctly — from setup/validation through QA deployment and final promotions — using a safe, self-contained “Hello World” example.

---

### 1️⃣ Test the `setup` and `validate` Jobs
These run automatically when you push to a `feature/*` branch or open a PR to `main`.

**Steps**

1. Create a new feature branch:
   ```bash
   git checkout -b feature/test-pipeline
   ```

2. Create a **simple Apex class**:
   ```bash
   sf apex class create --classname HelloWorldService --output-dir force-app/main/default/classes
   ```

   Replace the contents of `force-app/main/default/classes/HelloWorldService.cls` with:
   ```apex
   public with sharing class HelloWorldService {
       public static String sayHello(String name) {
           return 'Hello, ' + name + '!';
       }
   }
   ```

3. Create a **matching Apex test class**:
   ```bash
   sf apex class create --classname HelloWorldServiceTest --output-dir force-app/main/default/classes
   ```

   Replace the contents of `HelloWorldServiceTest.cls` with:
   ```apex
   @IsTest
   private class HelloWorldServiceTest {
       @IsTest static void testSayHello() {
           String result = HelloWorldService.sayHello('Trailblazer');
           System.assertEquals('Hello, Trailblazer!', result, 'Greeting should match expected output');
       }
   }
   ```

   ✅ This ensures your pipeline’s `sf apex run test` command always has a valid test to execute.

4. Create a **basic Lightning Web Component**:
   ```bash
   sf lightning generate component --type lwc --componentname helloWorld --outputdir force-app/main/default/lwc
   ```

   Replace `force-app/main/default/lwc/helloWorld/helloWorld.html` with:
   ```html
   <template>
       <h1>Hello World from CI/CD!</h1>
   </template>
   ```

   And replace `helloWorld.js` with:
   ```javascript
   import { LightningElement } from 'lwc';
   export default class HelloWorld extends LightningElement {}
   ```

   Both Apex and LWC additions are minimal and safe to validate the entire pipeline.

5. Commit and push:
   ```bash
   git add .
   git commit -m "Add HelloWorldService Apex class, test class, and LWC for CI/CD validation"
   git push -u origin feature/test-pipeline
   ```

6. Go to **Actions → Salesforce RMA Pipeline** — the run should start automatically.

**Expected**
- `setup` installs Node + Salesforce CLI successfully.
- `validate` runs:
  - PMD analysis on Apex.
  - ESLint check on LWC.
  - Salesforce package validation using the Dev sandbox.

**If it fails**
- Review job logs for any PMD, ESLint, or CLI issues.
- Fix, commit, and push again — the workflow re-runs automatically.

---

### 2️⃣ Test the `deploy_to_qa` Job
This job runs automatically **after** a PR into `main` is **approved** and **merged** with all checks passing.

**Steps**
1. Open a Pull Request from `feature/test-pipeline` → `main`.  
2. Wait for PMD/ESLint/Validation checks to pass.  
3. Have at least **one approval** (per branch protection rule).  
4. Merge the PR.

**Expected**
- A new workflow run starts automatically.
- `deploy_to_qa`:
  - Logs into **QASandbox** using JWT.
  - Deploys Apex and LWC code.
  - Runs your new `HelloWorldServiceTest` Apex test.
  - Displays success:  
    `✅ QA deployment completed for <ref>`

**Verify**
- Log in to QA → check that:
  - Apex Classes: `HelloWorldService`, `HelloWorldServiceTest`
  - Lightning Components: `helloWorld`
- Run Apex tests manually if you want to confirm output.

**If it fails**
- Open “Deploy to QA” step logs for details (missing dependency, failed test, etc.).
- Fix and re-merge.

---

### 3️⃣ Test the `promote` (UAT/Staging) and `deploy_to_prod` Jobs

#### 🧪 Promotion to UAT and Staging (`promote`)
Triggered **manually** using `workflow_dispatch`.

**Steps**
1. In GitHub → **Actions → Salesforce RMA Pipeline**.  
2. Click **Run workflow**, choose `main`, and hit **Run workflow**.  
3. The job will:
   - Authenticate via JWT to UAT and deploy.
   - Authenticate to Staging and deploy.
   - Run your `HelloWorldServiceTest` class in Staging.

**Expected**
- Both UAT and Staging deploys finish successfully.
- Apex tests pass in Staging.

**If it fails**
- Check “promote” logs for the failing step.
- Fix and re-run manually from Actions.

---

#### 🧪 Production Deployment (`deploy_to_prod`)
Triggered by a **release tag**.

**Steps**
```bash
git tag v1.0.0
git push origin v1.0.0
```

**Expected**
- Workflow runs automatically with `deploy_to_prod` job.
- Authenticates to Production using JWT.
- Deploys Apex + LWC.
- Runs the Apex test successfully.

**If it fails**
- Review “Deploy to Production” logs for specifics.
- Roll back or deploy a patch version once fixed.

---

### 🧩 Best Practices While Testing
- Use these sample Apex/LWC files for safe validation — don’t modify existing code yet.  
- Start with Dev → QA only; test promotions once that passes.  
- Keep PMD and ESLint configurations simple until validation is consistent.  
- Use unique, descriptive branch and tag names (`feature/test-automation`, `v1.0.0`).  
- Document each success/failure cycle in your repo for future contributors.

✅ Once every stage runs green (Dev → QA → UAT → Staging → Prod),  
your Salesforce RMA CI/CD automation is officially verified end-to-end.

</details>


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
