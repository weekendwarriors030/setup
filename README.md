# Setup

Setting up GitHub (GH) and Azure DevOps (ADO) access for migrations.

## Steps to get a GitHub PAT

https://github.com/settings/tokens

Settings -> Developer Settings -> Personal Access Tokens -> Tokens (classic)

Classic key is easier to set up with all permissions

## Steps to get SSH key

https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

Add the **public** key (`.pub`) to GitHub. Never share the private key (the file without `.pub`).

https://github.com/settings/keys

Settings -> SSH and GPG Keys

## Using GEI (GitHub Enterprise Importer) to migrate a repo from ADO to GH

GEI migrates the **repository** (code, branches, tags, history) from Azure DevOps to GitHub.
It does **not** convert pipelines to GitHub Actions and does **not** migrate secrets,
service connections, variable groups or environments. Those are set up again manually in GitHub.

### Prerequisites

- GitHub CLI (`gh`) installed and logged in: `gh auth login`, then check with `gh auth status`
- A GitHub **organization** as the target (a personal account is not a valid target for GEI;
  a free organization is enough for a test)
- The ADO organization, project and repo name
- Two Personal Access Tokens (PATs): one for ADO, one for GitHub (see below)

Only use test code or code you are allowed to move. Check with Security / the Data Privacy
Officer before migrating client code.

### Step 1: Create the Azure DevOps PAT

1. Go to `https://dev.azure.com/<ADO-ORG>`
2. Click the **User settings** icon (top right) -> **Personal access tokens**
3. Click **+ New Token**
4. Fill in:
   - Name: `gei-migration`
   - Organization: the ADO organization that contains the repo
   - Expiration: short, for example 7 days
   - Scopes: **Custom defined**, with at least (verify against the current GEI docs):
     - Code: Read
     - Identity: Read
     - Project and Team: Read
5. Click **Create** and copy the token immediately. It is shown only once.

If you cannot create tokens, PATs may be restricted by your organization. Ask IT / Security
for an approved route.

### Step 2: Create the GitHub PAT (classic)

GEI needs a **classic** token (fine-grained tokens are not supported, verify in the GEI docs).

1. Go to https://github.com/settings/tokens
2. Settings -> Developer Settings -> Personal Access Tokens -> Tokens (classic)
3. Click **Generate new token (classic)**
4. Name: `gei-migration`, expiration: short (for example 7 days)
5. Scopes (verify against the current GEI docs): `repo`, `admin:org`, `workflow`
6. Click **Generate token** and copy it immediately
7. If your organization uses SSO: click **Configure SSO** next to the token and authorize it for the organization

### Step 3: Install the GEI extension

```bash
gh extension install github/gh-ado2gh
```

On WSL, if you get an ICU error (`Couldn't find a valid ICU package`), run:

```bash
export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
```

Or install libicu: `sudo apt install -y libicu-dev`.

### Step 4: Set the PATs (outside any AI agent session)

Run this in a normal terminal, not inside Copilot or another agent. Tokens typed in an agent
session end up in its context.

Bash / WSL:

```bash
read -rs -p "ADO PAT: " ADO_PAT; export ADO_PAT; echo
read -rs -p "GitHub PAT: " GH_PAT; export GH_PAT; echo
```

PowerShell:

```powershell
$env:ADO_PAT = [System.Net.NetworkCredential]::new("", (Read-Host "ADO PAT" -AsSecureString)).Password
$env:GH_PAT  = [System.Net.NetworkCredential]::new("", (Read-Host "GitHub PAT" -AsSecureString)).Password
```

(Paste the token at the prompt. Nothing is shown while typing, that is expected.)

Alternatively use a `.env` file as in the section below. Make sure `.env` is in `.gitignore`
and never committed.

Check that they are set, without printing the values:

```bash
[ -n "$ADO_PAT" ] && echo "ADO_PAT is set"
[ -n "$GH_PAT" ] && echo "GH_PAT is set"
```

### Step 5: Run the migration

You do not need to clone the ADO repo first. GEI reads it directly from Azure DevOps.
Do **not** create the GitHub repo beforehand: GEI creates it, and fails if the name already exists.

```bash
gh ado2gh migrate-repo \
  --ado-org <ADO-ORG> \
  --ado-team-project <ADO-PROJECT> \
  --ado-repo <ADO-REPO> \
  --github-org <GH-ORG> \
  --github-repo <NEW-GH-REPO-NAME> \
  --target-repo-visibility private
```

PowerShell (single line):

```powershell
gh ado2gh migrate-repo --ado-org <ADO-ORG> --ado-team-project <ADO-PROJECT> --ado-repo <ADO-REPO> --github-org <GH-ORG> --github-repo <NEW-GH-REPO-NAME> --target-repo-visibility private
```

- `--ado-repo` must match the name in Azure DevOps exactly
- `--github-org` must be an existing GitHub organization
- `--github-repo` is a new name of your choice

### Step 6: Clean up

```bash
unset ADO_PAT GH_PAT
```

PowerShell:

```powershell
Remove-Item Env:ADO_PAT, Env:GH_PAT
```

Then revoke both PATs (ADO: Personal access tokens -> Revoke. GitHub: Tokens (classic) -> Delete).

### Step 7: Verify

Open `https://github.com/<GH-ORG>/<NEW-GH-REPO-NAME>` and check that the code, branches, tags
and `azure-pipelines.yml` are there.

### Common errors

| Error | Likely cause |
|---|---|
| 403 / not authorized | Missing PAT scope, or SSO not authorized for the token |
| Repo already exists | Choose another `--github-repo` name |
| ADO repo not found | Typo in `--ado-org`, `--ado-team-project` or `--ado-repo` |
| ICU / libicu error | See Step 3 (WSL) |

### What GEI does not migrate

- Pipeline definitions -> use GitHub Actions Importer (`gh actions-importer`) or a Copilot migration agent
- Secrets, service connections, variable groups, Key Vault links, environments and approvals -> recreate in GitHub (secrets, variables, environments, OIDC for Azure)
- Work items and wiki
- Branch policies -> recreate as GitHub rulesets

## Steps to run repo migration with gh ado2gh (using a .env file)

Set the env values:

```bash
cp .env.example .env
# Add PATs to .env (ADO_PAT=... and GH_PAT=...). Never commit .env.
export $(cat .env)
echo $ABC
```

This will set your PAT values correctly to run gh ado2gh.

```bash
ORG=weekendwarriors030
REPO=test1
gh ado2gh migrate-repo \
  --ado-org $ORG \
  --github-org $ORG \
  --ado-repo $REPO \
  --github-repo $REPO \
  --ado-team-project $REPO
```

## Other migration tools

ADO2GH is just one of many.
Most require a PAT, so once you can run ADO2GH, you can likely run the other commands as well.

## Security notes

- Do not paste PATs into chats, prompts or commits
- Use short expiry and minimum scopes, and revoke after use
- Run token-handling commands outside AI agent sessions
