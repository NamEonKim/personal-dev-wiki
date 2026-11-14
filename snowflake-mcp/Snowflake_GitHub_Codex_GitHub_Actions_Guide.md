# Snowflake, GitHub, OpenAI Codex, and GitHub Actions

This guide connects a Snowflake development workflow with GitHub, Snowflake CLI, OpenAI Codex, and GitHub Actions. It covers the separate authentication paths involved, a Snowflake Git repository clone, and a least-privilege workflow that fetches changes from GitHub.

> **Scope and safety:** Examples use placeholders only. Never commit private keys, passphrases, tokens, or other credentials. Use the Snowflake GitHub App or GitHub Actions OIDC where supported; use a narrowly scoped token or encrypted key only when needed. Do not use `ACCOUNTADMIN` for routine automation.

## 1. Understand the components

### OpenAI Codex and Snowflake CoCo

- **OpenAI Codex** is OpenAI's coding agent. Codex CLI can work with local project files and run tools such as `snow` when you configure them and authorize the actions. A project's `AGENTS.md` can give Codex repository-specific instructions; it does not grant Snowflake access or configure credentials.
- **Snowflake CoCo** (documented in the Snowflake Cortex Code documentation) is Snowflake's AI agent experience. Snowflake documents CoCo in Snowsight, as a desktop IDE, and as a CLI. It can use Snowflake connections and local files according to its configured tools and the user's Snowflake role.
- They are separate products. Codex does not become Snowflake CoCo by connecting to Snowflake, and Snowflake CoCo does not configure Codex.

### Two independent authentication paths

1. **GitHub Actions → Snowflake:** Prefer GitHub OIDC workload identity federation (WIF). It avoids a long-lived Snowflake private key in GitHub Secrets.
2. **Snowflake → GitHub:** A Snowflake Git repository clone authenticates to its remote Git provider. For GitHub.com, prefer the Snowflake GitHub App OAuth flow where it fits. A token-based Git integration is an alternative; keep its credential in a Snowflake Secret, not in the repository or workflow file.

An Actions job that runs `ALTER GIT REPOSITORY ... FETCH` only updates Snowflake's clone from the remote repository. It does not deploy or execute the repository's SQL. A Snowflake Git clone is optional if your pipeline only needs to check out and deploy files directly from the Actions runner.

## 2. Choose authentication for Snowflake CLI

Snowflake supports key-pair authentication as an alternative to password authentication. Do not assume that MFA makes password authentication fail in every situation; MFA requirements depend on the account's authentication policy and client flow. For unattended workloads, use a supported service authentication method such as WIF, OAuth, or key-pair authentication instead of storing a user's password.

For GitHub Actions, this guide uses WIF. For a local Snowflake CLI connection, key-pair authentication is one supported option. Snowflake CLI supports encrypted RSA private keys. Use a distinct key or identity for each workload where practical, and rotate or remove keys that are no longer needed.

### Generate a local encrypted RSA key pair

The commands below use a POSIX-compatible shell and OpenSSL (macOS, Linux, Git Bash, or WSL). Native Windows users should use an approved OpenSSL installation and protect the private-key file with Windows ACLs.

```bash
umask 077
mkdir -p ~/.ssh/snowflake

# Prompts for a passphrase; creates an encrypted PKCS#8 private key.
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 aes-256-cbc \
  -inform PEM -out ~/.ssh/snowflake/rsa_key.p8

# Prompts for the private-key passphrase and writes the public key.
openssl rsa -in ~/.ssh/snowflake/rsa_key.p8 \
  -pubout -out ~/.ssh/snowflake/rsa_key.pub

chmod 700 ~/.ssh/snowflake
chmod 600 ~/.ssh/snowflake/rsa_key.p8
```

Keep the private key and its passphrase in separate protected locations. Never print the private key into logs or commit either file. The public key is not a secret.

### Register the public key in Snowflake

Use an authorized administrative role to register a **named key pair** for the Snowflake user. Snowflake recommends named key pairs because they support key management, expiration, and role restrictions.

Extract the public-key body without its PEM delimiters:

```bash
grep -v "BEGIN PUBLIC KEY" ~/.ssh/snowflake/rsa_key.pub \
  | grep -v "END PUBLIC KEY" | tr -d '\n'
```

Register that public-key value using a placeholder in the SQL worksheet; do not paste a private key or passphrase:

```sql
ALTER USER <snowflake_user>
  ADD KEY PAIR <key_pair_name>
  PUBLIC_KEY = '<public_key_body>';
```

Verify the named key pair in Snowflake using `SHOW USER KEY PAIRS` and the current Snowflake syntax. For the legacy `RSA_PUBLIC_KEY` property, `DESC USER` exposes its fingerprint. Compare fingerprints if you troubleshoot a mismatch.

## 3. Connect Snowflake to a GitHub repository

Snowflake creates a Git repository clone that it can fetch from a remote HTTPS repository. This is separate from the local Git checkout and from GitHub Actions authentication.

### Recommended: Snowflake GitHub App OAuth

For GitHub.com, Snowflake documents a preconfigured GitHub App flow. Its example allows the `https://github.com` prefix; the GitHub App authorization controls which repositories are authorized:

```sql
CREATE API INTEGRATION <git_api_integration>
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com')
  API_USER_AUTHENTICATION = (TYPE = SNOWFLAKE_GITHUB_APP)
  ENABLED = TRUE;
```

Complete the Snowflake GitHub App authorization in the relevant Snowflake interface, then create the clone with the repository's HTTPS URL:

```sql
CREATE GIT REPOSITORY <database>.<schema>.<repository_object>
  API_INTEGRATION = <git_api_integration>
  ORIGIN = 'https://github.com/<github_owner>/<repository>.git';

ALTER GIT REPOSITORY <database>.<schema>.<repository_object> FETCH;
SHOW GIT BRANCHES IN <database>.<schema>.<repository_object>;
```

This Snowflake GitHub App flow is documented for GitHub.com, including standard GitHub Enterprise Cloud organizations hosted there. GitHub Enterprise Cloud with data residency and GitHub Enterprise Server require a different OAuth setup; confirm current Snowflake support before configuring them.

### Alternative: token-based Git integration

Use this only when the GitHub App OAuth flow is unsuitable. GitHub recommends fine-grained personal access tokens (PATs) over classic PATs when the use case supports them. Limit a token to the one repository, grant only the read access needed to fetch source (for example, Contents: read), set an expiration, and follow any organization approval policy. If the required GitHub integration cannot use a fine-grained token, use the narrowest approved alternative and plan rotation.

Create a Snowflake Secret using an approved secure administrative process. Do not put an actual token in a checked-in SQL file, a workflow, a terminal transcript, or a chat message. Example shape only:

```sql
CREATE SECRET <database>.<schema>.<github_secret>
  TYPE = password
  USERNAME = '<github_username>'
  PASSWORD = '<token supplied through an approved secure process>';
```

Then restrict the API integration to the required GitHub owner prefix and explicitly allow only that Secret:

```sql
CREATE API INTEGRATION <git_api_integration>
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/<github_owner>')
  ALLOWED_AUTHENTICATION_SECRETS = (<database>.<schema>.<github_secret>)
  ENABLED = TRUE;
```

Create the Git repository clone with `GIT_CREDENTIALS = <database>.<schema>.<github_secret>`. The Secret must be in the integration's allowed authentication secrets. Prefer `CREATE` over `CREATE OR REPLACE` for first-time setup; replacing an existing object can disrupt a working integration.

### Grant least-privilege access

Have the appropriate Snowflake administrator create the API integration and grant only the privileges needed to create or fetch the clone. Avoid granting broad database, schema, or account roles to make setup easier.

```sql
GRANT USAGE ON DATABASE <database> TO ROLE <git_setup_role>;
GRANT USAGE ON SCHEMA <database>.<schema> TO ROLE <git_setup_role>;
GRANT CREATE GIT REPOSITORY ON SCHEMA <database>.<schema>
  TO ROLE <git_setup_role>;
GRANT USAGE ON INTEGRATION <git_api_integration>
  TO ROLE <git_setup_role>;
```

For token authentication, also grant `USAGE` on the Secret to the role that creates the Git repository clone. To allow a separate Actions role to run `FETCH`, grant `WRITE` on the Git repository (or assign ownership only if that is necessary and approved):

```sql
GRANT USAGE ON DATABASE <database> TO ROLE <git_sync_role>;
GRANT USAGE ON SCHEMA <database>.<schema> TO ROLE <git_sync_role>;
GRANT USAGE ON WAREHOUSE <warehouse> TO ROLE <git_sync_role>;
GRANT WRITE ON GIT REPOSITORY <database>.<schema>.<repository_object>
  TO ROLE <git_sync_role>;
```

## 4. Install and configure Snowflake CLI locally

Install Snowflake CLI using Snowflake's standalone installer or a supported package manager. For example, with `uv`:

```bash
uv tool install snowflake-cli
snow --version
```

Add a local connection interactively:

```bash
snow connection add
```

Use your Snowflake account identifier, username, least-privilege role, and warehouse. For key-pair authentication, select `SNOWFLAKE_JWT` and specify the private-key file path. Do not save the key passphrase in a shared configuration file; use a protected local secret store or the CLI's supported passphrase environment variable. Keep local Snowflake connection files out of source control.

Test a named connection:

```bash
snow connection test -c <connection_name>
```

Use the connection for a reviewed SQL file only when you are authorized to run it in the selected environment:

```bash
snow sql -c <connection_name> -f <sql_file>
```

## 5. Fetch the Snowflake Git clone from GitHub Actions

### Configure GitHub OIDC and the Snowflake service user

Configure a Snowflake service user with a workload identity whose issuer is `https://token.actions.githubusercontent.com` and whose subject exactly matches the intended GitHub Actions workflow identity. Restrict the subject to the repository and protected branch or environment that should run the workflow. Assign only the `<git_sync_role>` role to this user.

For example, the Snowflake service-user definition has this shape; replace the subject with the exact claim from your GitHub OIDC configuration:

```sql
CREATE USER <github_actions_service_user>
  TYPE = SERVICE
  WORKLOAD_IDENTITY = (
    TYPE = OIDC
    ISSUER = 'https://token.actions.githubusercontent.com'
    SUBJECT = '<exact_github_actions_subject>'
  )
  DEFAULT_ROLE = <git_sync_role>;

GRANT ROLE <git_sync_role> TO USER <github_actions_service_user>;
```

The Actions subject varies by trigger and can be customized. Do not copy a sample subject into production without checking the actual workflow claims. Snowflake's `snowflake-actions` supports OIDC with Snowflake CLI 3.11 or later.

Set non-secret account and user identifiers as GitHub repository variables if desired. OIDC removes the need to store a Snowflake password or private key in GitHub Secrets. Protect the default branch and restrict who can change the workflow.

### Add the workflow

This workflow fetches the Snowflake clone after a push to `main` or when manually dispatched. It does not execute SQL files. The repository object name, account, and service user are configuration values, not credentials.

```yaml
name: Fetch Snowflake Git repository

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

concurrency:
  group: snowflake-git-fetch
  cancel-in-progress: false

jobs:
  fetch:
    runs-on: ubuntu-latest
    steps:
      - name: Set up Snowflake CLI with OIDC
        uses: snowflakedb/snowflake-actions@v3
        with:
          cli-version: "3.20.0"
          use-oidc: true

      - name: Fetch remote changes into Snowflake
        env:
          SNOWFLAKE_ACCOUNT: ${{ vars.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ vars.SNOWFLAKE_USER }}
          SNOWFLAKE_ROLE: ${{ vars.SNOWFLAKE_ROLE }}
          SNOWFLAKE_WAREHOUSE: ${{ vars.SNOWFLAKE_WAREHOUSE }}
        run: |
          snow connection test -x
          snow sql -x -q "ALTER GIT REPOSITORY <DATABASE>.<SCHEMA>.<REPOSITORY_OBJECT> FETCH;"
```

Configure those GitHub variables with non-secret values and assign the role the required database, schema, warehouse, and Git repository privileges. Confirm the Snowflake CLI and Action versions against current release documentation before upgrading or pinning them. For stricter supply-chain controls, pin GitHub Actions to reviewed full commit SHAs. Do not expose OIDC permissions to untrusted pull-request code, and do not add `pull_request_target` for this workflow.

If WIF is unavailable, configure a dedicated service user's encrypted key pair and store the private-key material and passphrase as separate, protected GitHub environment secrets. Restrict the workflow to a protected branch/environment, write the key file without interpolating secrets into the script body, set owner-only file permissions, and remove the temporary file when the job ends. Prefer WIF instead.

## 6. Give OpenAI Codex safe project instructions

Codex reads `AGENTS.md` files as project instructions. Keep the file concise and repository-specific; credentials and access must still be configured separately. For example:

```markdown
# AGENTS.md

## Project context
- This repository contains Snowflake SQL and data-pipeline code.
- Use the `snow` CLI only with a configured, least-privilege connection.
- Default to the development database and schema unless the task says otherwise.

## Safe workflow
- Inspect the working tree and the target environment before changing files or running SQL.
- Review generated SQL and diffs before execution.
- Never print, commit, or copy credentials, private keys, tokens, or passphrases.
- Do not run destructive or production-changing SQL without explicit authorization.
- Do not commit or push changes automatically; prepare them for review.
```

Codex instructions do not grant Snowflake privileges. The Snowflake role, account policy, and CLI connection determine what a command can access. Avoid instructions that tell an agent to push straight to `main` or run destructive SQL without review.

## 7. Troubleshoot common issues

| Symptom | What to check |
|---|---|
| Key-pair login fails | Confirm the registered public key matches the private key, the CLI connection uses `SNOWFLAKE_JWT`, the private-key path is correct, and the passphrase is available through a protected channel. |
| Password or MFA challenge in automation | Confirm the workflow uses its intended service identity and supported non-interactive method (prefer WIF/OIDC); do not disable MFA or reuse a human password to work around policy. |
| Git API integration rejects the repository | Compare the HTTPS origin with the integration's allowed prefix, check that the integration is enabled, and confirm the selected GitHub authentication method. |
| `Object does not exist` or `not authorized` during `FETCH` | Confirm the active role has `USAGE` on the database/schema and `WRITE` or `OWNERSHIP` on the Git repository. For token auth, verify the Secret is allowed by the API integration and visible to the required role. |
| OIDC authentication fails | Check `id-token: write`, the Snowflake CLI version, service-user type, issuer, exact subject claim, account identifier, and assigned role. Restrict the subject rather than broadening it to all branches. |
| Workflow succeeds but SQL changes are not applied | `FETCH` only refreshes Snowflake's Git clone. Add a separate, reviewed deployment step if the goal is to execute or deploy SQL. |

## Official documentation

- [Snowflake key-pair authentication and key rotation](https://docs.snowflake.com/en/user-guide/key-pair-auth)
- [Snowflake workload identity federation](https://docs.snowflake.com/en/user-guide/workload-identity-federation)
- [Snowflake Git setup and authentication](https://docs.snowflake.com/en/developer-guide/git/git-setting-up-public)
- [Snowflake Git repository overview](https://docs.snowflake.com/en/developer-guide/git/git-overview)
- [CREATE GIT REPOSITORY](https://docs.snowflake.com/en/sql-reference/sql/create-git-repository)
- [ALTER GIT REPOSITORY and access requirements](https://docs.snowflake.com/en/sql-reference/sql/alter-git-repository)
- [Snowflake CLI installation](https://docs.snowflake.com/en/developer-guide/snowflake-cli/installation)
- [Snowflake CLI connections](https://docs.snowflake.com/en/developer-guide/snowflake-cli/connecting/configure-connections)
- [Snowflake Actions for GitHub Actions](https://github.com/snowflakedb/snowflake-actions)
- [Snowflake CoCo (Cortex Code) overview](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code)
- [OpenAI Codex `AGENTS.md` instructions](https://developers.openai.com/codex/guides/agents-md)
- [GitHub personal access token guidance](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [Using secrets in GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [GitHub Actions OIDC reference](https://docs.github.com/en/actions/reference/security/oidc)

*Documentation checked: 2026-11-14. Snowflake, GitHub, and client features change; verify the linked vendor documentation and your account policies before rollout.*
