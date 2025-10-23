# Salesforce Unlocked Package (2GP) with Source Control

This guide walks through setting up a **Salesforce 2nd Generation Unlocked Package (2GP)** and connecting it to **GitHub** for modern DevOps workflows.  
It’s formatted for clean viewing and publication in repositories.

---

<details>
<summary><strong>▶ Level 1: Getting to Know Salesforce Unlocked Package (2GP)</strong></summary>

This section explains how to set up and practice modular, source-driven development using **Salesforce CLI**, leveraging **Dev Hub** for package management and **Scratch Orgs** for testing.

### Setup and Authentication
Start from your **Dev Hub org** and enable **Dev Hub** under:  
**Setup → Dev Hub → Enable Dev Hub**  
(Optional) Also enable **Second-Generation Packaging**.

### Authenticate with Salesforce CLI:
```bash
sf org login web --set-default-dev-hub --alias DevHub
sf org list
```

### Create Scratch Org
```bash
sf org create scratch --definition-file config/project-scratch-def.json --alias myScratch --duration-days 7 --set-default
sf org list
```

### Create Unlocked Package
```bash
sf package create --name "MyApp" --path force-app --package-type Unlocked --description "Practice Package"
```

### Create Package Version
```bash
sf package version create --package "MyApp" --installation-key-bypass --wait 10
sf package version list
```
Take note of the **Subscriber Package Version ID** (starts with `04t`).

### Install Package in Scratch Org
```bash
sf package install --package "MyApp@1.0.0-1" --target-org myScratch --wait 10 --noprompt
sf package installed list --target-org myScratch
```

### Open Scratch Org
```bash
sf org open --target-org myScratch
```

</details>

---

<details>
<summary><strong>▶ Level 2: Getting to Know GitHub with Salesforce Unlocked Package (2GP)</strong></summary>

This section guides you through connecting your Salesforce project to **GitHub** for source control and collaboration.  
You’ll learn how to create a GitHub account, set up your repository, authenticate securely with a **Personal Access Token**, and push your local source code to GitHub.

### 1. Create a Free GitHub Account
If you don’t already have one: [https://github.com/join](https://github.com/join)

### 2. Create a New Repository on GitHub
1. Log in to your GitHub account  
2. Click **New Repository**  
3. **Name it exactly the same as your local Salesforce project folder** (e.g., `MyApp`)  
   - Keeps local and remote project structures consistent  
4. Choose **Public** or **Private** visibility  
5. Leave it **empty** (no README or `.gitignore`)  
6. Click **Create Repository**  

> ⚙️ When you run `git remote add origin` from inside your local project folder, that folder becomes the root of your Git repository — it won’t be nested one level deeper.

Copy the HTTPS URL (e.g., `https://github.com/<your-username>/<your-repo>.git`).

### 3. Generate a Personal Access Token (for Authentication)
1. Go to **Settings → Developer settings → Personal access tokens → Tokens (classic)**  
   or directly [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. Click **Generate new token → Generate new token (classic)**
3. Name it **"VS Code Git Access"**
4. Under **Scopes**, select:  
   - `repo` – for access to your repositories  
   - `workflow` *(optional)* – if you plan to use GitHub Actions later
5. Click **Generate token** and **copy it immediately** — it will only be visible once.

### 4. Install and Configure Git
```bash
git --version
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### 5. Initialize a Local Git Repository
```bash
cd path/to/your/project
git init
echo ".sfdx/" >> .gitignore
echo ".sf/" >> .gitignore
echo "node_modules/" >> .gitignore
echo "*.log" >> .gitignore
git add .
git commit -m "Initial commit – Salesforce 2GP project"
```

### 6. Link and Push Your Local Project to GitHub
```bash
git remote add origin https://github.com/<your-username>/<your-repo>.git
git branch -M main
git push -u origin main
```

### 7. Verify the Connection
```bash
git remote -v
```

### 8. Work with Branches (Feature Development Flow)
To make isolated updates or build new components, create and switch to a **feature branch**:
```bash
git checkout -b feature/new-component
```

Make your changes and commit them as you go.  
If you’re modifying unlocked package components, create a **new minor version** before committing.  
Since your current version is **0.1.0**, the next logical minor version should be **0.2.0**.

**Update `sfdx-project.json`:**
```json
"versionName": "ver 0.2",
"versionNumber": "0.2.0.NEXT"
```
> ⚙️ The `packageAliases` section updates automatically after version creation.

**Create the new package version:**
```bash
sf package version create --package "MyApp" --installation-key-bypass --wait 10
sf package version list
```

**Commit and push your updates:**
```bash
git add .
git commit -m "Added new component and created 0.2 minor package version"
git push -u origin feature/new-component
```

### 9. Review and Track Changes
```bash
git status
git log --oneline --graph
```

> ✅ **Tip:** Personal Access Tokens are safer than passwords and can be revoked anytime from GitHub → Settings → Developer Settings → Personal Access Tokens.

</details>

---

<details>
<summary><strong>▶ Level 3: Creating a Dependent Package (“People”) for MyApp</strong></summary>

Goal: **Create a separate Salesforce SFDX project and GitHub repository** for the **People** package, which adds a single field (`Nick Name`) to the `Contact` object.  
Then make **MyApp** depend on it by creating a new **Project__c** object to reference Contact and display its Nick Name.  

---

### 1️⃣ Create People Project and Repository
```bash
sf project generate --name People
cd People
git init
git add .
git commit -m "Initialize People SFDX project"
git branch -M main
git remote add origin https://github.com/<your-username>/People.git
git push -u origin main
```

### 2️⃣ Configure `sfdx-project.json`
```json
{
  "packageDirectories": [
    {
      "path": "force-app",
      "default": true,
      "package": "People",
      "versionName": "ver 0.1",
      "versionNumber": "0.1.0.NEXT"
    }
  ],
  "namespace": "",
  "sourceApiVersion": "60.0",
  "packageAliases": {}
}
```

### 3️⃣ Add Contact.Nick_Name__c Field
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

Commit and push:
```bash
git add .
git commit -m "Add Nick Name field"
git push
```

### 4️⃣ Build People Package
```bash
sf package create --name "People" --path force-app --package-type Unlocked --description "Shared Contact field"
sf package version create --package "People" --installation-key-bypass --wait 10
sf package version list
```

### 5️⃣ Add Dependency in MyApp
Edit `sfdx-project.json`:
```json
"dependencies": [{ "package": "People", "versionNumber": "0.1.0.LATEST" }]
```

### 6️⃣ Create Project__c Object and Fields
**Object**
```bash
mkdir -p force-app/main/default/objects/Project__c/fields
touch force-app/main/default/objects/Project__c/Project__c.object-meta.xml
```
```xml
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
  <label>Project</label>
  <pluralLabel>Projects</pluralLabel>
  <sharingModel>ReadWrite</sharingModel>
</CustomObject>
```

**Lookup (Contact)**
```bash
touch force-app/main/default/objects/Project__c/fields/Contact__c.field-meta.xml
```
```xml
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
  <fullName>Contact__c</fullName>
  <label>Contact</label>
  <referenceTo>Contact</referenceTo>
  <type>Lookup</type>
</CustomField>
```

**Formula (Nick Name)**
```bash
touch force-app/main/default/objects/Project__c/fields/Contact_Nick_Name__c.field-meta.xml
```
```xml
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
  <fullName>Contact_Nick_Name__c</fullName>
  <label>Contact Nick Name</label>
  <formula>Contact__r.Nick_Name__c</formula>
  <type>Text</type>
</CustomField>
```

### 7️⃣ Build New MyApp Version
```bash
sf package version create --package "MyApp" --installation-key-bypass --wait 10
sf package version list
```

### 8️⃣ Validate in Scratch Org
```bash
sf org create scratch --definition-file config/project-scratch-def.json --alias depScratch --duration-days 7 --set-default
sf package install --package "People@0.1.0-1" --wait 10 --noprompt --target-org depScratch
sf package install --package "MyApp@0.3.0-1" --wait 10 --noprompt --target-org depScratch
sf org open --target-org depScratch
```

### 9️⃣ Commit and Push to Feature Branch
After successful validation, commit and push your **MyApp** updates to the same feature branch created earlier:
```bash
git add .
git commit -m "Added dependent People package and Project__c reference changes"
git push origin feature/new-component
```

### 🔟 Create a Pull Request
Go to your GitHub **MyApp** repository → you’ll see a prompt to create a **Pull Request**.  
Click **Compare & pull request**, review your changes, and submit the PR to merge `feature/new-component` → `main`.

---

### ✅ Summary
- **People** → separate package/repo.  
- **MyApp** → depends on People.  
- **Project__c** → links Contact and shows Nick Name.  
- Continue work under `feature/new-component`, push changes, and open a PR to `main`.  
- Install order: People → MyApp.

</details>

---

<details>
<summary><strong>▶ Gotchas & Common Pitfalls</strong></summary>

### Quick Summary of Package Versioning Best Practice

`MAJOR.MINOR.PATCH.BUILD`

| Option | Type | Example | Use |
|:-------|:-----|:---------|:----|
| **A** | Major | `2.0.0.NEXT` | Breaking changes |
| **B** | Minor | `1.1.0.NEXT` | New features |
| **C** | Patch | `1.0.1.NEXT` | Fixes/tweaks |
| **D** | Build | `1.0.0.NEXT` | Successive test builds |

#### 🔹 Best Practice Notes
- Update the `versionNumber` in **packageDirectories** before creating a new version.
- Let CLI handle updates to **packageAliases** automatically.
- Tag Git after each release (`git tag v1.1.0 && git push origin v1.1.0`).
- Promote tested versions only:
```bash
sf package version promote --package "MyApp@<version>" --noprompt
```

### Professional Notes

- **Dev Hub** is the central registry for all 2GP packages and versions.  
- Always version from committed source to ensure traceability.  
- **Scratch Orgs** are short-lived; use them for isolated testing.  
- Use sandboxes for longer validation or user acceptance testing.  
- **Unlocked Packages** enable modular, repeatable, version-controlled deployments.  
- Integrate these steps into **GitHub Actions**, **Jenkins**, or **Azure Pipelines** for continuous delivery.

</details>
