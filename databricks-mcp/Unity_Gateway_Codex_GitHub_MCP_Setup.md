# Connect Codex to GitHub MCP Through Unity Gateway (Windows)

This guide walks through the setup in Windows PowerShell, from the prerequisites
to testing the connection.

> **Goal:** Sign in to Databricks Unity Gateway and register the
> `system.ai.github` MCP server with Codex.
>
> **Privacy:** The examples below do not contain a real user email address,
> Windows account name, or Databricks workspace identifier. Replace each
> `<...>` placeholder with a value from your own environment.

---

## 1. Prerequisites

- Windows 10 or 11
- PowerShell
- Python 3.12 or later and `uv`
- A Databricks workspace account and permission to access that workspace
- Permission to use the `system.ai.github` MCP server in the workspace
- Codex installed and ready to run

**Important:** Unity Gateway's `ug mcp add` command registers MCP settings for
Codex. If you use Codex Desktop, fully quit and restart the app after
registering the server.

## 2. Install `uv`

Open PowerShell and run:

```powershell
winget install --id astral-sh.uv -e
```

When installation finishes, close PowerShell completely and open a new window.

Verify the installation:

```powershell
uv --version
```

Example output:

```text
uv 0.x.x
```

If PowerShell does not recognize `uv`, first make sure you opened a new window.
If it still is not recognized, check the installation and PATH configuration.

## 3. Install Unity Gateway

Install Unity Gateway with:

```powershell
uv tool install git+https://github.com/databricks/unity-gateway
```

Verify the installation:

```powershell
uv tool list
ug --help
```

Installation is successful when Unity Gateway appears in `uv tool list` and
`ug --help` displays its help.

If PowerShell cannot find `ug`, open a new PowerShell window and try again.

## 4. Install the Databricks CLI

Unity Gateway invokes the `databricks` command when it starts Databricks
authentication, so you also need the Databricks CLI.

```powershell
winget install Databricks.DatabricksCLI
```

Verify the installation:

```powershell
databricks -v
```

Example output:

```text
Databricks CLI v1.x.x
```

### Why is the CLI required?

If the Databricks CLI is missing, `ug mcp add` may fail with an error such as:

```text
FileNotFoundError: [WinError 2]
```

In that case, check whether the Databricks CLI is installed and available on
PATH before reinstalling Unity Gateway.

## 5. Sign in to your Databricks workspace

Replace the workspace address in this command with your own Databricks
workspace URL:

```powershell
databricks auth login --host "https://<DATABRICKS_WORKSPACE_HOST>"
```

For example, if your workspace address has the form
`https://<workspace-id>.cloud.databricks.com`, use that address as-is.

If a browser opens during sign-in, log in to your Databricks account and
authorize access. If the terminal asks for a profile name, choose an easy-to-
remember name. You do not need to use your email address as the profile name.

Check authentication:

```powershell
databricks auth profiles
```

Alternatively, check the authentication status for a specific profile:

```powershell
databricks auth describe --profile "<DATABRICKS_PROFILE_NAME>"
```

## 6. Register the GitHub MCP server with Codex

Replace the workspace address in this command with your own:

```powershell
ug mcp add --agents codex --workspace "https://<DATABRICKS_WORKSPACE_HOST>" --names system.ai.github
```

The options mean:

- `ug mcp add`: Add a Unity Gateway MCP server to an agent's configuration
- `--agents codex`: Register it with Codex
- `--workspace`: Specify the Databricks workspace to use
- `--names system.ai.github`: Select the GitHub MCP server

For a first-time setup, you may be prompted to sign in to Databricks. If you
are already authenticated, the existing authentication is reused.

On success, output similar to the following is displayed:

```text
Databricks auth already available
Add MCP Servers
Configuring for: Codex
Saved
```

## 7. Check the MCP registration

Run:

```powershell
ug mcp list --agents codex
```

Example of a successful registration:

```text
NAME              LOCATION          AGENTS  STATUS
system-ai-github  system.ai.github  codex   enabled
```

- `system-ai-github`: The server name shown in the configuration list
- `system.ai.github`: The full name of the Databricks MCP service
- `codex`: The server is registered for Codex
- `enabled`: The configuration is enabled

**Note:** `enabled` means the configuration was saved and enabled. It does not
guarantee that GitHub API calls or repository access will work. Test an actual
tool call.

## 8. Find the Codex configuration file

Codex stores MCP server settings in `config.toml`.

Default Windows user configuration path:

```text
%USERPROFILE%\.codex\config.toml
```

Open it in Notepad from PowerShell:

```powershell
notepad "$env:USERPROFILE\.codex\config.toml"
```

If the file does not exist yet, Codex or the configuration tool may create it.
`ug mcp add --agents codex` registers MCP settings for Codex automatically, so
do not manually add or edit entries as a first step.

Codex MCP settings commonly use a TOML table like this:

```toml
[mcp_servers.example]
command = "example-command"
args = ["--example"]
```

This is **only an example to explain the format**; it is not a replacement for
the GitHub MCP settings generated by Unity Gateway. Do not overwrite
auto-generated entries without a reason.

If you use project-specific settings, you can also use `.codex/config.toml` in
the project folder. Use project configuration only for projects you trust.

## 9. Test the connection in Codex

1. Fully quit Codex Desktop.
2. Restart Codex Desktop.
3. Check whether GitHub-related tools appear in the MCP tool list or Codex's
   connected-tools view.
4. Request an actual tool call, for example:

   ```text
   Use the GitHub MCP tools to list the GitHub repositories
   I am authorized to access. Do not make any changes.
   ```

The connection test succeeds if a tool call runs and returns a list of
repositories you can access.

**Important:** A natural-language response from Codex alone does not confirm
that MCP is connected. Verify that a GitHub MCP tool was actually called.

## 10. Troubleshooting

### A. PowerShell does not recognize `uv`

Example error:

```text
uv : The term 'uv' is not recognized as the name of a cmdlet...
```

Try these steps:

1. Run `winget install --id astral-sh.uv -e`.
2. Close PowerShell completely.
3. Open a new PowerShell window.
4. Check with `uv --version`.

### B. PowerShell does not recognize `ug`

Check:

```powershell
uv tool list
uv tool dir --bin
```

If Unity Gateway is not installed, install it again:

```powershell
uv tool install git+https://github.com/databricks/unity-gateway
```

Then open a new PowerShell window and run `ug --help` again.

### C. `FileNotFoundError: [WinError 2]`

The Databricks CLI may not be found. Install it:

```powershell
winget install Databricks.DatabricksCLI
```

Open a new PowerShell window and check:

```powershell
databricks -v
```

Then sign in to Databricks.

### D. Databricks authentication error

```powershell
databricks auth login --host "https://<DATABRICKS_WORKSPACE_HOST>"
```

Confirm that you completed browser authentication, signed in to the correct
workspace, and have permission to access that workspace.

### E. The server is `enabled`, but Codex does not show its tools

1. Check the registration with `ug mcp list --agents codex`.
2. Check `%USERPROFILE%\.codex\config.toml`.
3. Fully quit and restart Codex Desktop.
4. Check Codex's MCP tool list.
5. Confirm that your Databricks workspace account has permission to use the
   MCP server.

## 11. Command summary

Run these commands in order for a fresh setup:

```powershell
# 1. Install uv
winget install --id astral-sh.uv -e

# 2. Verify uv (in a new PowerShell window)
uv --version

# 3. Install Unity Gateway
uv tool install git+https://github.com/databricks/unity-gateway

# 4. Verify Unity Gateway
ug --help

# 5. Install the Databricks CLI
winget install Databricks.DatabricksCLI

# 6. Verify the Databricks CLI
databricks -v

# 7. Sign in to Databricks
databricks auth login --host "https://<DATABRICKS_WORKSPACE_HOST>"

# 8. Register GitHub MCP with Codex
ug mcp add --agents codex --workspace "https://<DATABRICKS_WORKSPACE_HOST>" --names system.ai.github

# 9. Check the registration
ug mcp list --agents codex
```

---

## Official documentation

- Unity Gateway project: https://github.com/databricks/unity-gateway
- Databricks — Connect Codex:
  https://docs.databricks.com/aws/en/ai-gateway/coding-agent-codex
- Databricks — Add MCP tools and skills:
  https://docs.databricks.com/aws/en/ai-gateway/coding-agent-mcp-skills
- OpenAI — Codex MCP configuration:
  https://learn.chatgpt.com/docs/extend/mcp?surface=cli

## Security and privacy

- Do not publish workspace URLs, email addresses, usernames, tokens, OAuth
  information, or personal access tokens (PATs) in a public repository.
- `config.toml` may contain authentication tokens or other sensitive values.
  Review and redact secrets before sharing the file.
- Redact email addresses, workspace identifiers, tokens, and profile names
  from screenshots or error logs before sharing them.
- The GitHub MCP server's actual read/write scope depends on Databricks MCP
  permissions and the scope of the GitHub connection and authorization.
