#  Complete CICD setup - Github Actions

---

### CI/CD Setup — Step 1: Preparing SSH Access for GitHub Actions

**Purpose:**
GitHub Actions needs a way to securely connect to the production server to pull the latest code and rebuild containers on every push. Instead of reusing a personal SSH key, a dedicated key pair was generated solely for CI/CD access this way, access can be revoked independently without affecting other login methods.

**Steps performed:**

1. Generated a new SSH key pair on the server itself:
   ```bash
   ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github-actions-deploy
   ```
   This creates two files inside `~/.ssh/` on the server:
   - `github-actions-deploy` — the **private** key
   - `github-actions-deploy.pub` — the **public** key

2. Added the **public** key to the server's own `authorized_keys` file, since the key was generated locally on the same machine it needs to authenticate against:
   ```bash
   cat ~/.ssh/github-actions-deploy.pub >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

3. Displayed the **private** key so it could be copied into GitHub Secrets:
   ```bash
   cat ~/.ssh/github-actions-deploy
   ```
   Copied the entire output (including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----` lines) — this goes into the `SSH_PRIVATE_KEY` GitHub Secret in the next step, and is never committed to the repo.

4. Verified the key works for local (loopback) SSH login before wiring it into GitHub Actions:
   ```bash
   ssh -i ~/.ssh/github-actions-deploy ubuntu@localhost
   ```
   Confirmed login succeeded without a password prompt.

**Why a dedicated key instead of the server's existing login key:**
- If the CI/CD key is ever leaked or needs to be rotated, it can be removed from `authorized_keys` without affecting personal SSH access to the server
- Limits the blast radius — this key's only purpose is running automated deploy commands, nothing else

---

Here's Step 2 — adding GitHub Secrets so the workflow can use the SSH key securely.

## Step 2: Add GitHub Secrets

1. Go to your repo on GitHub
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret** and add each of these one at a time:

| Secret Name | Value |
|---|---|
| `SERVER_HOST` | `<public ip>` |
| `SERVER_USER` | `ubuntu` |
| `SSH_PRIVATE_KEY` | paste the full private key you got from `cat ~/.ssh/github-actions-deploy` |
| `SERVER_PORT` | `22` |

4. Save each one — GitHub will show them as `***` (hidden) once saved; you won't be able to view the value again, only overwrite it

Once you've added these, let me know and send me a screenshot or just confirm the secret names you used (not the values) — I'll write up the documentation for this step, then we move to Step 3 (creating the workflow file).

## Step 3: Create the Workflow File

1. In your project root, create the following folder structure (GitHub Actions specifically looks for this path):
   ```
   Real-Time-WebSocket-Application-/
                              └── .github/
                                     └── workflows/
                                            └── deploy.yml
   ```

2. Create the file:
   ```bash
   mkdir -p .github/workflows
   nano .github/workflows/deploy.yml
   ```

   ```

## Step 4: Define the Deploy Job

1. Open the workflow file again:
   ```bash
   nano .github/workflows/deploy.yml
   ```

2. Add a `jobs` section below the trigger, using the `appleboy/ssh-action` to connect to your server and run the deploy commands:
   ```yaml
   name: Deploy to Production

   on:
     push:
       branches: [main]

   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - name: Deploy to server via SSH
           uses: appleboy/ssh-action@v1.0.0
           with:
             host: ${{ secrets.SERVER_HOST }}
             username: ${{ secrets.SERVER_USER }}
             key: ${{ secrets.SSH_PRIVATE_KEY }}
             port: ${{ secrets.SERVER_PORT || 22 }}
             script: |
               cd ~/Real-Time-WebSocket-Application-
               git pull origin main
               docker-compose down
               docker-compose up -d --build
   ```

3. A few things to get right here:
   - `cd ~/Real-Time-WebSocket-Application-`
   - `${{ secrets.SERVER_HOST }}` etc. pull directly from the GitHub Secrets
   - `docker-compose down` before `up -d --build` ensures old containers are fully removed before rebuilding (avoids stale container name conflicts); you can drop `down` if you want faster, near-zero-downtime deploys, since `up -d --build` alone will recreate changed containers

4. Save, commit, and push:
   ```bash
   git add .github/workflows/deploy.yml
   git commit -m "Add SSH deploy job to CI/CD workflow"
   git push
   ```

5. Go to the **Actions** tab on GitHub and watch the workflow run — click into it to see live logs of the SSH connection and deploy commands executing
