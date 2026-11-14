# Postman API Testing and OAuth Token Automation

Organize REST API requests, test responses, and automate API-specific access-token workflows with Postman.

## Organize collections and environments

Postman workspaces hold collaborative work. Collections group related requests, and folders can organize workflows such as authentication, users, and orders. Each request defines an HTTP method, URL, headers, authorization, and optional body. Use a collection-level authorization setting when its requests share the same scheme; set child requests to inherit it and override only when an endpoint requires different authentication.

Use environments for configuration that changes between local, test, and production systems. Keep a base URL such as `https://api.example.com` in an environment and reference it as `{{base_url}}`. Use collection variables for values shared by one collection but not tied to an environment. Global variables have the broadest scope and are best kept for temporary prototyping.

When names collide, Postman resolves values from the narrowest applicable scope (local, data, environment, collection, then global). Prefer specific scopes to avoid a request silently using a value from a broader scope.

## Send and verify REST requests

1. Select the method and URL, for example `GET {{base_url}}/v1/users`.
2. Enter query parameters in **Params** rather than manually concatenating an encoded query string.
3. For a JSON `POST`, `PUT`, or `PATCH`, choose **Body > raw > JSON** and provide the API's expected payload:

   ```json
   {
     "username": "sample-user",
     "email": "user@example.com"
   }
   ```

4. Send the request and verify the status, headers, body, and response time against the API contract. A `204 No Content` response, for example, has no body to inspect.
5. Add assertions in the request's **Scripts > Post-response** area to make checks repeatable:

   ```javascript
   pm.test("returns a successful response", function () {
     pm.expect(pm.response.code).to.be.within(200, 299);
   });

   pm.test("returns a JSON body", function () {
     pm.response.to.have.header("Content-Type");
   });
   ```

   Adapt assertions to the endpoint contract; not every successful endpoint returns JSON or a `200` status.

Run a collection or folder with the Collection Runner to execute requests in order and review test results. Collection runs can also use iteration data. For CI/CD, evaluate the Postman CLI and the secret-injection options supported by your runner.

## Authentication and safe variable handling

For a bearer-token API, select **Bearer Token** in the collection's **Authorization** tab and use a variable for the token. Postman adds the `Bearer` prefix; enter the token value, not a second prefix. Set child requests to **Inherit auth from parent**. If the API uses another authentication scheme, configure the scheme required by that API instead.

An OAuth access token is sent to a resource server to authorize API calls. A refresh token, when the authorization server issues one, is sent to its token endpoint to request a replacement access token. Lifetimes, response fields, rotation, revocation, and refresh-token reuse rules are determined by the API provider; do not assume fixed durations or that every refresh response contains a new refresh token.

Treat tokens, API keys, passwords, and client secrets as credentials:

- Never put real credentials in documentation, examples, exported collections, source control, or screenshots. Do not log token values in scripts or the Postman Console.
- Do not place sensitive values in shared or synced environments, collection variables, or globals. Masking a value in the UI does not by itself mean it is private or unsynced.
- Prefer Postman Vault or an organization-approved secrets manager for secrets. Postman Local Vault secrets are not synced to Postman Cloud; Shared Vault has workspace access controls and is shared by design. Check plan availability, access scope, and the behavior of your chosen run mode.
- If an API-specific workflow stores a short-lived token in an environment variable, use a local, unshared environment and do not export or sync its sensitive value. Use placeholders in shared templates.
- Grant only the access needed, and revoke or rotate credentials according to the provider's policy.

Variable values are available to scripts as strings. Use the scope appropriate for the workflow, such as `pm.environment`, `pm.collectionVariables`, or `pm.globals`; do not use a global variable for a secret merely for convenience.

## Save tokens from a login response

If the API's login response contains token fields, a **Post-response** script can extract them for subsequent requests. Adjust field names and the response shape to the actual API. This example writes to the selected environment; use it only in a local, unshared environment when the values are sensitive.

```javascript
if (pm.response.code < 200 || pm.response.code >= 300) {
  throw new Error(`Login failed with HTTP ${pm.response.code}`);
}
const body = pm.response.json();
const accessToken = body.accessToken;
if (typeof accessToken !== "string" || accessToken.length === 0) {
  throw new Error("Login response did not contain accessToken");
}

pm.environment.set("access_token", accessToken);

// Only set this when the API returns a new refresh token.
if (typeof body.refreshToken === "string" && body.refreshToken.length > 0) {
  pm.environment.set("refresh_token", body.refreshToken);
}
```

In the collection's Authorization tab, set the bearer token to `{{access_token}}`. For APIs that nest fields (for example, under `data`), read the documented response path instead. Avoid saving a guessed expiration time: use an expiry value returned by the API, or follow its documented policy.

## Refresh a token

The simplest workflow is a separate refresh request, run when needed:

1. Create a `POST` request to the API's documented token-refresh endpoint.
2. Configure the authorization method required by that endpoint. Some APIs require **No Auth** and accept the refresh token in a JSON body; others require client authentication or form parameters.
3. Supply the refresh token using the API's documented field and content type. The endpoint URL, grant parameters, and response shape are API-specific.
4. In a Post-response script, validate the response and save the new access token. If the provider rotates refresh tokens, replace the old refresh token only when a new one is returned; follow the provider's rules for invalidating the old token.

An optional collection **Pre-request** script can refresh before a request when an expiry timestamp is available. The following illustrates the control flow, not a universal OAuth implementation. Replace the endpoint, request body, and response properties with the API's documented contract. Use a local environment for the variables and avoid running concurrent requests that could race while a rotating refresh token is being replaced.

```javascript
const refreshToken = pm.environment.get("refresh_token");
const expiresAt = Number(pm.environment.get("access_token_expires_at"));
const accessToken = pm.environment.get("access_token");
const refreshEarlyByMs = 30_000;
const needsRefresh =
  !accessToken ||
  (Number.isFinite(expiresAt) && Date.now() >= expiresAt - refreshEarlyByMs);

if (needsRefresh) {
  if (!refreshToken) {
    throw new Error("Cannot refresh: refresh_token is not set");
  }

  const response = await pm.sendRequest({
    url: `${pm.environment.get("base_url")}/v1/auth/refresh`,
    method: "POST",
    header: { "Content-Type": "application/json" },
    body: {
      mode: "raw",
      raw: JSON.stringify({ refreshToken })
    }
  });

  if (response.code < 200 || response.code >= 300) {
    throw new Error(`Token refresh failed with HTTP ${response.code}`);
  }

  const body = response.json();
  if (typeof body.accessToken !== "string" || body.accessToken.length === 0) {
    throw new Error("Refresh response did not contain accessToken");
  }
  pm.environment.set("access_token", body.accessToken);

  if (typeof body.refreshToken === "string" && body.refreshToken.length > 0) {
    pm.environment.set("refresh_token", body.refreshToken);
  }
  const expiresIn = Number(body.expiresIn);
  if (
    body.expiresIn !== undefined &&
    body.expiresIn !== null &&
    Number.isFinite(expiresIn) &&
    expiresIn > 0
  ) {
    pm.environment.set(
      "access_token_expires_at",
      String(Date.now() + expiresIn * 1000)
    );
  }
}
```

This example expects an `expiresIn` duration in seconds and a JSON response; change it if the API provides a different expiration field. If expiry is unknown, the script cannot reliably infer it from the token unless the API explicitly documents that approach. Failures should prompt the operator to inspect the response and re-authenticate when required; do not silently continue with a stale token.

Postman's built-in OAuth 2.0 authorization can request and refresh tokens interactively when supported by the provider. Review the token-sharing controls before syncing a token. Postman's OAuth 2.0 authentication is not supported by scheduled runs, monitors, the Postman CLI, or Newman in the same way as interactive requests; the current documentation notes that a synced token may be needed and automatic token refresh is not supported in those modes. Confirm current product and plan behavior before relying on it in automation.

## References

- [Postman: Store and reuse values using variables](https://learning.postman.com/docs/use/send-requests/variables/variables/)
- [Postman: Store secrets in Postman Vault](https://learning.postman.com/docs/use/postman-vault/postman-vault-secrets/)
- [Postman: API authentication and authorization](https://learning.postman.com/docs/use/send-requests/authorization/authorization/)
- [Postman: OAuth 2.0 authentication](https://learning.postman.com/docs/use/send-requests/authorization/oauth-20/)
- [Postman: Use scripts to send requests](https://learning.postman.com/docs/tests-and-scripts/write-scripts/postman-sandbox-reference/pm-send-request/)
- [Postman: Test your API using the Collection Runner](https://learning.postman.com/docs/tests-and-scripts/running-collections/intro-to-collection-runs/)
