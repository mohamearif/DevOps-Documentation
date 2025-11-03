# Salesforce DevOps – JWT Setup Guide (Collapsible Edition)

This guide explains how to configure **JWT-based authentication** for Salesforce CLI and GitHub Actions.  
You’ll create RSA keys, set up a Connected App, authorize a user, and store credentials in GitHub Secrets.  
This enables secure, password-free automation across all your Salesforce environments.

---

<details>
<summary><strong>▶ 1️⃣ Generate RSA Key Pair (First Step)</strong></summary>

Salesforce uses asymmetric encryption (RSA) for the **JWT Bearer Flow**.  
You create two files:
- `server.key` → private key (kept secret)
- `server.crt` → public certificate (uploaded to Salesforce)

Run these commands inside a folder you’ll keep safe, e.g., `jwt-keys/`.

### 🔹 Step 1: Create a folder for keys
```bash
mkdir jwt-keys && cd jwt-keys
```
This creates a **jwt-keys** folder and moves you into it.

### 🔹 Step 2: Generate a private RSA key
```bash
openssl genrsa -out server.key 2048
```
This generates a **2048-bit private RSA key** and saves it as `server.key`.  
> ⚠️ Keep this file secret. Never check it into GitHub.

### 🔹 Step 3: Create a public certificate
```bash
openssl req -x509 -new -key server.key -out server.crt -days 365 -subj "/CN=GitHubCI"
```
Explanation:
- Uses the private key to create a **self-signed X.509 certificate** (`server.crt`).
- `-days 365` makes it valid for 1 year.
- `/CN=GitHubCI` adds a friendly label for identification.
- This file (`server.crt`) is the public key you’ll upload to Salesforce.

### 🔹 Step 4: Verify the files
```bash
ls -l
```
You should see:
```
server.key
server.crt
```

✅ You now have:
- **server.key** → private key for signing JWTs (kept secret)  
- **server.crt** → public certificate to upload to Salesforce

</details>

---

<details>
<summary><strong>▶ 2️⃣ Create a Connected App in Salesforce</strong></summary>

1. Log in to your Salesforce org (e.g., **DevSandbox**) and go to  
   **Setup → App Manager → New Connected App**

2. Fill in:
   - **Connected App Name:** `GitHub CI Integration`
   - **API Name:** auto-filled
   - **Contact Email:** your email

3. Under **API (Enable OAuth Settings)**:
   - ✅ Check **Enable OAuth Settings**
   - **Callback URL:**  
     `https://login.salesforce.com/services/oauth2/callback`  
     *(or for Sandboxes: `https://test.salesforce.com/services/oauth2/callback`)*

4. Under **Selected OAuth Scopes:**
   - `Manage user data via APIs (api)`
   - `Perform requests on your behalf (refresh_token, offline_access)`

5. ✅ Check **Use Digital Signatures**

6. Upload the **public key (`server.crt`)** created earlier.

7. Click **Save**, then copy the **Consumer Key (Client ID)**.  
   You’ll use this later as `SF_CLIENT_ID_DEV`, `SF_CLIENT_ID_QA`, etc.

> ⚠️ If it doesn’t appear immediately, open the app → “Manage” → “Edit Policies” →  
> Set **Permitted Users** to *Admin approved users are pre-authorized*.

</details>

---

<details>
<summary><strong>▶ 3️⃣ Grant Access to the Integration User</strong></summary>

1. In **Setup → Profiles** (or **Permission Sets**):  
   open the profile of your **integration user** — the one CI/CD will use.

2. Under **Connected App Access**, add the Connected App created above.

3. Save changes.

✅ The integration user is now allowed to log in using your Connected App via JWT.

</details>

---

<details>
<summary><strong>▶ 4️⃣ Test JWT Login Locally</strong></summary>

Use Salesforce CLI to confirm JWT authentication works.

```bash
sf org login jwt \
  --username your.integration.user@example.com \
  --client-id YOUR_CONNECTED_APP_CLIENT_ID \
  --jwt-key-file jwt-keys/server.key \
  --alias DevSandbox
```

If successful, you’ll see:
```
Successfully authorized DevSandbox
```

Verify:
```bash
sf org list
```

✅ If your org alias appears and is active, JWT login is successful.

</details>

---

<details>
<summary><strong>▶ 5️⃣ Store Secrets in GitHub</strong></summary>

Now that JWT works locally, store your credentials securely for automation.

1. Go to your GitHub repo →  
   **Settings → Secrets and variables → Actions → New repository secret**

2. Add the following (repeat per environment):

| Secret | Description |
|:--|:--|
| `SF_JWT_KEY` | Content of your `server.key` (private key) |
| `SF_CLIENT_ID_DEVHUB` | Connected App Client ID for the Dev Hub (org that owns the 2GP packages) |
| `SF_USERNAME_DEVHUB` | Integration user email for the Dev Hub |

3. Repeat for other environments (QA, UAT, Staging, Prod), e.g.:
```
SF_CLIENT_ID_QA
SF_USERNAME_QA
```

✅ GitHub Actions will use these secrets for JWT login in pipelines.

</details>

---

<details>
<summary><strong>▶ ✅ Summary</strong></summary>

| Step | Purpose | Output |
|:--|:--|:--|
| **Generate RSA keys** | Create secure private/public key pair | `server.key`, `server.crt` |
| **Create Connected App** | Register JWT connection | App with uploaded `server.crt` |
| **Grant Access** | Authorize integration user | User linked to Connected App |
| **Test Login** | Confirm CLI access | JWT authentication verified |
| **Add to GitHub** | Enable pipeline access | Secrets for CI/CD pipelines |

---

### 🧩 Best Practices
- Use **separate Connected Apps** for Dev, QA, UAT, Staging, and Prod.  
- **Rotate RSA keys** annually before expiration.  
- Keep the **private key (`server.key`)** off GitHub and shared drives.  
- Use least-privilege access for your integration user.  
- For sandboxes, prefer the `test.salesforce.com` login endpoint.

✅ Once complete, your CI/CD pipelines (e.g., `SFDevOps-RMA-V2.1.md`) can authenticate securely without manual logins.
</details>
