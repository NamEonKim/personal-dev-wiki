# Snowflake MCP Setup Guide

This guide explains how to configure Snowflake’s documented **Snowflake-managed MCP server** and connect it to an MCP-compatible client. It uses Snowflake’s official documentation for the server URL, SQL object, access control, and OAuth setup. Examples use placeholders only; never commit account identifiers, client secrets, tokens, or personal credentials.

> **Scope:** The managed MCP server is a Snowflake service. You create an MCP server object in Snowflake; you do not install a local Snowflake MCP server process. Snowflake documents client connections through the managed endpoint and OAuth. This guide does not claim that third-party or community MCP implementations are Snowflake-supported.

## 1. Prerequisites

- A Snowflake account and a region where the Snowflake-managed MCP server is available. Review the current regional availability and limitations in the official documentation.
- A database and schema in which an authorized role can create the MCP server object.
- At least one existing Snowflake resource to expose, such as a Cortex Agent, Cortex Search service, Cortex Analyst semantic view, SQL execution capability, UDF, or stored procedure.
- A client that supports MCP over the documented remote endpoint and Snowflake OAuth or the configured External OAuth provider.
- An administrator or security owner to configure OAuth and grant only the required privileges.

The Snowflake-managed MCP server currently supports tools, including tool discovery and invocation. It does not implement every MCP capability; check the official limitations before relying on resources, prompts, roots, or other protocol features.

## 2. Create a least-privilege MCP server

For governed business-data access, Snowflake recommends exposing a Cortex Agent as the client-facing tool. The agent can orchestrate its approved Analyst, Search, or custom tools; the client then connects to one governed interface.

The following is the official `CREATE MCP SERVER` form with a placeholder agent:

```sql
CREATE MCP SERVER <database_name>.<schema_name>.<server_name>
  FROM SPECIFICATION $$
  tools:
    - title: "Governed business data agent"
      name: "business_data_agent"
      type: "CORTEX_AGENT_RUN"
      identifier: "<database_name>.<schema_name>.<agent_name>"
      description: "Answers governed business data questions."
  $$;
```

Use precise tool names and descriptions. Snowflake also documents other tool types, including Cortex Search, Cortex Analyst, SQL execution, and generic UDF/stored-procedure tools. Configure only the tools the intended client needs. Tool annotations are hints for clients, not security controls.

## 3. Grant access to the server and its resources

Use a dedicated role, grant only the required privileges, and assign that role to authorized users. For example, an agent-backed server may require:

```sql
CREATE ROLE <mcp_access_role>;

GRANT DATABASE ROLE SNOWFLAKE.CORTEX_AGENT_USER TO ROLE <mcp_access_role>;
GRANT USAGE ON WAREHOUSE <warehouse_name> TO ROLE <mcp_access_role>;
GRANT USAGE ON DATABASE <database_name> TO ROLE <mcp_access_role>;
GRANT USAGE ON SCHEMA <database_name>.<schema_name> TO ROLE <mcp_access_role>;
GRANT USAGE ON MCP SERVER <database_name>.<schema_name>.<server_name>
  TO ROLE <mcp_access_role>;
GRANT USAGE ON AGENT <database_name>.<schema_name>.<agent_name>
  TO ROLE <mcp_access_role>;
GRANT ROLE <mcp_access_role> TO USER <username>;
```

Grant the role only the underlying object privileges its agent tools require. For example, a Search tool needs `USAGE` on its Cortex Search service; an Analyst tool can require `SELECT` on its semantic view and underlying data; generic tools require the documented privilege on their UDF or stored procedure. Access to the MCP server alone does not grant access to every tool. Do not grant broad access to `PUBLIC` or all current/future tables merely to make a client work.

## 4. Configure OAuth

Snowflake-managed MCP servers use OAuth 2.0. Snowflake OAuth is the default; External OAuth can be configured to use an organization’s identity provider. The MCP server does **not** support dynamic client registration.

Create a Snowflake OAuth security integration using the client type and exact redirect URI required by your client. Snowflake documents `CONFIDENTIAL` clients for clients that can safely store a client secret and `PUBLIC` clients for clients using PKCE without a client secret. Do not guess the redirect URI: use the exact callback URI displayed in the client’s current setup instructions.

Example shape for a confidential client:

```sql
CREATE SECURITY INTEGRATION <integration_name>
  TYPE = OAUTH
  OAUTH_CLIENT = CUSTOM
  ENABLED = TRUE
  OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
  OAUTH_REDIRECT_URI = '<exact_client_redirect_uri>'
  OAUTH_USE_SECONDARY_ROLES = NONE
  ALLOWED_ROLES_LIST = ('<mcp_access_role>');
```

For a public client using PKCE, follow the current Snowflake instructions for `OAUTH_CLIENT_TYPE = 'PUBLIC'` and the client’s exact redirect URI. Restrict `ALLOWED_ROLES_LIST` to the intended access role. Snowflake recommends leaving secondary roles disabled for a least-privilege MCP setup unless broader behavior is deliberately required.

After creating the integration, an appropriately authorized administrator can retrieve client credentials with the official system function:

```sql
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('<INTEGRATION_NAME>');
```

The integration name passed to the function is case sensitive and must be uppercase. Treat its output as a secret: do not paste it in a guide, source control, tickets, terminal transcripts, or chat. Store it in the MCP client’s approved secret store.

If you use External OAuth, configure an External OAuth security integration and bind it at account, database, or schema scope using `OAUTH_AUTHORIZATION_SERVER`. Follow Snowflake’s issuer, audience, claim-mapping, and role-scope requirements; do not copy example IdP values as real configuration.

## 5. Find the MCP endpoint

Snowflake documents this endpoint format:

```text
https://<account_url>/api/v2/databases/<database>/schemas/<schema>/mcp-servers/<name>
```

Use your account’s correct Snowflake URL and the fully qualified database, schema, and server names. The account identifier format is documented by Snowflake. Some clients have hostname issues with underscores; Snowflake recommends hyphens for MCP connection hostnames. Do not invent a region URL or substitute a URL from another account.

## 6. Connect a client

Snowflake documents registration in MCP-compatible clients. The exact UI and OAuth callback vary by client and can change over time.

### Claude custom connector

Snowflake’s current common-client instructions include Claude (Claude.ai and Claude Desktop):

1. In Claude, open **Settings → Connectors**.
2. Select **Add custom connector** (or browse for Snowflake if available).
3. Enter a connector name and the MCP endpoint from the previous section.
4. Provide the OAuth client ID and secret from the Snowflake integration only through the client’s secure setup flow.
5. Set the OAuth redirect URI in Snowflake to the exact URI Claude displays for that client. The callback differs between Claude.ai and Claude Desktop.
6. Complete the browser-based Snowflake sign-in and consent, then confirm the expected tools are visible.

For other clients, follow their current official instructions for a remote MCP endpoint and OAuth. This guide intentionally does not provide speculative JSON/TOML client configuration because the schema, OAuth support, and callback requirements are client-specific.

## 7. Validate and troubleshoot

| Symptom | Check |
|---|---|
| OAuth consent or connection fails | Confirm the redirect URI exactly matches the current client callback and that the integration is enabled. |
| Authentication succeeds but server connection fails | Check the complete endpoint, account URL, database, schema, and server name. |
| Hostname connection error | Check for underscores in the account hostname; use the documented hyphen form where applicable. |
| Session initialization fails | Confirm the user has a default warehouse and the access role has `USAGE` on it. |
| Wrong role/data scope | Verify the user’s default role and the OAuth integration’s `ALLOWED_ROLES_LIST`; ensure secondary roles are not enabling unintended access. |
| No tools are listed | Grant `USAGE` on the MCP server and, for agent tools, `USAGE` on the referenced agent. Check the user’s effective role. |
| A listed tool fails when called | Grant the least-privilege underlying object access required by that specific agent, Search service, semantic view, function, or procedure. |
| PrivateLink SaaS client cannot connect | Snowflake instructs SaaS MCP clients to use the public MCP endpoint and configure the authorization endpoint for PrivateLink as documented. |
| Remote SaaS client is blocked by a network policy | Permit the client provider’s published outbound IPs according to Snowflake’s network-policy instructions. Do not guess IP ranges. |

You can inspect server objects with `SHOW MCP SERVERS` and `DESCRIBE MCP SERVER`; remove an object only when its owner has approved the change. Review Snowflake’s current troubleshooting table for additional cases.

## Security checklist

- Prefer OAuth over hardcoded tokens. Keep OAuth client secrets in an approved secret manager or the client’s secure credential store.
- Use a dedicated least-privileged role and a dedicated warehouse; do not use administrative roles for routine MCP access.
- Grant privileges on the MCP server and each underlying tool separately; test which tools are visible and callable as the intended user.
- Verify third-party MCP servers and their tool descriptions before enabling them; tool descriptions can be misleading.
- Treat tool annotations as client hints, not authorization. Enforce access with Snowflake roles and object grants.
- Review client outbound IP allowlists and private connectivity requirements with the security team.
- Avoid recursive agent/MCP call paths that can generate unbounded or expensive work.
- Recheck availability, supported clients, OAuth behavior, and limitations in current Snowflake documentation before production rollout.

## Official documentation

- [Snowflake-managed MCP server](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp)
- [Snowflake key concepts and architecture](https://docs.snowflake.com/en/user-guide/intro-key-concepts)
- [Snowflake OAuth overview](https://docs.snowflake.com/en/user-guide/oauth-snowflake-overview)
- [Snowflake account identifiers](https://docs.snowflake.com/en/user-guide/admin-account-identifier)
- [CREATE SECURITY INTEGRATION (Snowflake OAuth)](https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-oauth-snowflake)
- [CREATE SECURITY INTEGRATION (External OAuth)](https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-oauth-external)
- [Snowflake network policies](https://docs.snowflake.com/en/user-guide/network-policies)

*Documentation checked: 2026-11-14. Snowflake and client features can change; consult the linked vendor documentation before deployment.*
