# Business Central Online Integration Guide (Postman)

This note explains how other teams can consume the Business Central (BC) Online APIs using the shared Postman collection. It assumes you already have access to the Postman workspace link: https://web.postman.co/workspace/a751533a-c80d-4e4e-847e-c4a5cf3309ec.

## What you get

- A Postman collection with the BC Online endpoints exposed by the Staff Portal API (mission note, imprest, payment lines, approval, etc.).
- Environment-variable driven URLs so you can point to any BC Online environment without editing every request.
- OAuth2 (Azure AD) bearer-token auth flow prewired for BC.

## Prerequisites

- BC Online tenant with the target environment (e.g., `Production`, `Sandbox`).
- AAD app registration with client credentials allowed.
- API permissions on the app: `Dynamics 365 Business Central` (delegated or application) and admin consent granted.
- Postman installed or a Postman web account (recommended: web, since the collection link opens in the browser).
- Company name (as seen in BC) and, if using per-company endpoints, the company ID or the URL-encoded company name.

## One-time Azure AD app setup (if you do not have one)

1. Go to Azure Portal > App registrations > New registration.
2. Name it (e.g., `BC-Online-Integration`) and keep `Accounts in this org directory`.
3. Redirect URI is not required for client credentials.
4. After creation, capture:
   - Application (client) ID
   - Directory (tenant) ID
5. Certificates & secrets > New client secret. Capture the secret value now.
6. API permissions > Add permission > `Dynamics 365 Business Central` > choose `application` or `delegated` as needed > `API.ReadWrite.All` (or the minimal scope you need) > Grant admin consent.

## Postman collection import

1. Open the workspace link above in Postman. If you cannot access it, request access from the collection owner.
2. Click **Fork** or **Add to Workspace** to bring the collection into your own workspace.
3. Also add the accompanying Postman environment (if shared). If not shared, create a new environment with the variables below.

## Postman environment variables

Use these names to match the collection placeholders (adjust if your fork uses different keys):

| Variable       | Example                                                            | Notes                                                  |
| -------------- | ------------------------------------------------------------------ | ------------------------------------------------------ |
| `tenantId`     | `contoso.onmicrosoft.com` or GUID                                  | Your AAD tenant ID.                                    |
| `environment`  | `Production` or `Sandbox`                                          | BC environment name.                                   |
| `bcBaseUrl`    | `https://api.businesscentral.dynamics.com/v2.0`                    | Base for BC Online API.                                |
| `companyName`  | `My Company`                                                       | URL-encoded if it has spaces (Postman can URL-encode). |
| `clientId`     | GUID                                                               | From the AAD app registration.                         |
| `clientSecret` | secret value                                                       | Store as **Current value** only.                       |
| `scope`        | `https://api.businesscentral.dynamics.com/.default`                | Default BC scope.                                      |
| `tokenUrl`     | `https://login.microsoftonline.com/{{tenantId}}/oauth2/v2.0/token` | AAD token endpoint.                                    |
| `accessToken`  | (fetched at runtime)                                               | Set by the auth request; reused by other calls.        |

If your collection uses company IDs instead of names, add `companyId` and substitute in the URLs accordingly.

## Getting a bearer token in Postman

1. In the collection, open the `Auth` or `Get Token` request (client-credentials flow).
2. Body (x-www-form-urlencoded):
   - `grant_type=client_credentials`
   - `client_id={{clientId}}`
   - `client_secret={{clientSecret}}`
   - `scope={{scope}}`
3. Send the request. On success you receive `access_token` and `expires_in`.
4. Set a test script (already present in the shared collection) to write `access_token` into the environment variable `accessToken`. If absent, add:
   ```javascript
   const jsonData = pm.response.json();
   pm.environment.set("accessToken", jsonData.access_token);
   ```
5. Subsequent requests should use **Authorization: Bearer {{accessToken}}** (often set at the collection level).

## Calling BC Online endpoints

Common URL patterns you will see in the collection:

- Standard BC API root: `{{bcBaseUrl}}/{{tenantId}}/{{environment}}/api/v2.0/companies`
- Company-scoped custom API/page (example): `{{bcBaseUrl}}/{{tenantId}}/{{environment}}/api/v2.0/companies({{companyId}})/pagestaffportalapi`
- If the collection uses URL-encoded company names: `.../companies({{companyName}})` (Postman can encode automatically).

Tips:

- Always URL-encode company names with spaces or special characters.
- Keep `environment` exactly as it appears in BC (case-sensitive in URLs).
- For date/time fields, send ISO 8601 with offset (e.g., `2025-12-16T00:00:00Z`).

## Typical flow

1. Set environment variables.
2. Run `Auth` to populate `accessToken`.
3. Exercise the mission note/imprest/payment endpoints in the collection (create header, add lines, submit for approval, etc.).
4. Repeat `Auth` when the token expires (usually 3600 seconds for client credentials).

## Troubleshooting

- **401/403**: Confirm `accessToken` is fresh, scope is `.default`, and admin consent was granted.
- **404**: Check company name/ID and environment in the URL. Ensure the extension is published in that environment.
- **400 date parsing**: Provide `DateTimeOffset` values with `Z` or an explicit offset.
- **Permission denied** inside BC: Verify the AAD app has a BC user mapped with the correct permission set.

## Hand-off checklist for new teams

- Access to the Postman workspace/collection granted.
- AAD app registration (client ID/secret + consent) available.
- Environment variables in Postman filled with tenant, environment, company, and auth details.
- Token request tested successfully.
- One or two sample business calls executed (e.g., create/update mission note, add lines, submit for approval).

For any issues, include the request URL, status code, and response body when asking for help.
