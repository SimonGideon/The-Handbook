# Business Central Online Integration Guide (Postman)

This note explains how teams can consume the Business Central (BC) Online APIs using a shared Postman collection. The collection and credentials (tenant, environment, client ID/secret, company identifiers) are provided separately; ensure you receive them through a secure channel before testing.

## What you get

- A Postman collection with BC Online endpoints (custom API page actions such as create header, add lines, submit for approval, etc.).
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

1. Open the shared collection you received (out-of-band). If you cannot access it, request access from the collection owner.
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
- Company-scoped custom API/page (example): `{{bcBaseUrl}}/{{tenantId}}/{{environment}}/api/v2.0/companies({{companyId}})/page<yourapipage>`
- If the collection uses URL-encoded company names: `.../companies({{companyName}})` (Postman can encode automatically).

Tips:

- Always URL-encode company names with spaces or special characters.
- Keep `environment` exactly as it appears in BC (case-sensitive in URLs).
- For date/time fields, send ISO 8601 with offset (e.g., `2025-12-16T00:00:00Z`).

## Typical flow

1. Set environment variables.
2. Run `Auth` to populate `accessToken`.
3. Exercise a few sample endpoints in the collection (e.g., create a document header, add lines, submit for approval).
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

## Using a custom API page (AL)
Extend your API page (PageType = API) with service-enabled procedures. Follow the usual pattern: return JSON as `Text` and keep endpoints under your chosen `EntitySetName` (example below uses `myOperations`).

Example procedures to list, save, and delete Items via the StaffPortalAPI page:

```al
// In your API page (PageType = API)
[ServiceEnabled]
procedure GetItems() result: Text
var
   ItemRec: Record Item;
   Response: JsonObject;
   Data: JsonArray;
   One: JsonObject;
begin
   Clear(Data);
   if ItemRec.FindSet() then
      repeat
         Clear(One);
         One.Add('id', ItemRec."No.");
         One.Add('description', ItemRec.Description);
         One.Add('blocked', ItemRec.Blocked);
         Data.Add(One);
      until ItemRec.Next() = 0;

   Response.Add('success', true);
   Response.Add('message', 'Items retrieved successfully');
   Response.Add('data', Data);
   Response.Add('timestamp', Format(CurrentDateTime, 0, 9));
   Response.WriteTo(result);
end;

[ServiceEnabled]
procedure SaveItem(id: Code[20]; description: Text; blocked: Boolean) result: Text
var
   ItemRec: Record Item;
   Response: JsonObject;
   DataJson: JsonObject;
begin
   if ItemRec.Get(id) then begin
      ItemRec.Description := description;
      ItemRec.Blocked := blocked;
      ItemRec.Modify(true);
   end else begin
      ItemRec.Init();
      ItemRec.Validate("No.", id);
      ItemRec.Description := description;
      ItemRec.Blocked := blocked;
      ItemRec.Insert(true);
   end;

   Clear(DataJson);
   DataJson.Add('id', id);
   Response.Add('success', true);
   Response.Add('message', 'Item saved successfully');
   Response.Add('data', DataJson);
   Response.Add('timestamp', Format(CurrentDateTime, 0, 9));
   Response.WriteTo(result);
end;

[ServiceEnabled]
procedure DeleteItem(id: Code[20]) result: Text
var
   ItemRec: Record Item;
   Response: JsonObject;
   DataJson: JsonObject;
begin
   if ItemRec.Get(id) then
      ItemRec.Delete(true);

   Clear(DataJson);
   DataJson.Add('id', id);
   Response.Add('success', true);
   Response.Add('message', 'Item deleted successfully');
   Response.Add('data', DataJson);
   Response.Add('timestamp', Format(CurrentDateTime, 0, 9));
   Response.WriteTo(result);
end;
```

Key reminders:

- Keep to one API page per bounded context to maintain a consistent surface.
- Return JSON `Text` responses using `JsonObject`.
- Validate fields before insert/modify and avoid exposing sensitive columns.

## Publishing the web service for OData

Publishing the extension exposes API pages automatically. Verify in BC **Web Services** that the page appears; the OData URL follows:

`{{bcBaseUrl}}/{{tenantId}}/{{environment}}/api/{{publisher}}/{{group}}/{{version}}/companies({{companyId}})/{{entitySetName}}`

## Calling the API and procedures (OData v4)

Entity CRUD (standard OData):

- List: `GET .../myOperations`
- Get by id: `GET .../myOperations({{id}})`
- Create: `POST .../myOperations` with `{ "id": "ITEM-001", "description": "Demo" }`
- Update: `PATCH .../myOperations({{id}})` with `{ "description": "Updated" }`

Bound actions (service-enabled procedures on your API page):

- Get items (returns list): `POST .../myOperations/Microsoft.NAV.GetItems` with body `{}`
- Save item (upsert): `POST .../myOperations/Microsoft.NAV.SaveItem` with body `{ "id": "ITEM-001", "description": "Demo", "blocked": false }`
- Delete item: `POST .../myOperations/Microsoft.NAV.DeleteItem` with body `{ "id": "ITEM-001" }`

Note: replace `myOperations` with your `EntitySetName` and ensure the action names match your `[ServiceEnabled]` procedures.

## API page configuration example

Example header configuration for a custom API page that hosts `[ServiceEnabled]` actions:

```al
page 50000 MyApiPage
{
    APIGroup = 'mygroup';
    APIPublisher = 'mycompany';
    APIVersion = 'v1.0';
    Caption = 'My API';
    EntityName = 'myOperation';
    EntitySetName = 'myOperations';
    PageType = API;
    SourceTable = "Portal API Request";
    SourceTableTemporary = true;
    ODataKeyFields = "Entry No.";
    DelayedInsert = true;
    InsertAllowed = false;
    DeleteAllowed = false;
    ModifyAllowed = false;
    Extensible = false;
}
```

Why this setup matters:

- **EntitySetName** drives the root segment of all endpoints, e.g., `.../companies({companyId})/myOperations/Microsoft.NAV.<Action>`.
- **Temporary source table + dummy row**: Keep the page backed by a temporary table and insert a harmless dummy row in `OnOpenPage` so the entity set is not empty and OData discovery works consistently.
- **`ODataKeyFields`**: Provides an addressable key for the entity set; actions remain unbound and don’t require a specific record ID.
- **UI fields**: Expose only safe fields (e.g., `SystemId`, `Entry No.`) for diagnostics; keep business data in the actions.

## Portal API Request table structure (example)

Sample lightweight logging table you can adapt (use any object ID/namespace you prefer):

```al
table "Portal API Request"
{
   DataClassification = CustomerContent;
   TableType = Temporary;

   fields
   {
      field(1; "Entry No."; Integer)
      {
         AutoIncrement = true;
      }
      field(2; "Operation"; Code[50]) { }
      field(3; "Parameters"; Text[2048]) { }
      field(4; "Response Success"; Boolean) { }
      field(5; "Response Message"; Text[250]) { }
      field(6; "Response Data"; Text[2048]) { }
   }

   keys
   {
      key(PK; "Entry No.") { Clustered = true; }
   }
}
```

Notes:

- Mark both the table and the API page as temporary to keep dummy rows out of persistent storage.
- Add Blob fields, correlation IDs, or timestamps if you need deeper auditing; keep the base schema lean for performance.
- `SystemId` is still available automatically for diagnostics.

## Calling from Postman (safe pattern)

- Use the company-scoped URL pattern above.
- Keep `clientSecret` and `accessToken` only in **Current Value**; do not export secrets.
- Scrub tenant/company IDs before sharing screenshots or examples.

## Call examples (no secrets)

Replace placeholders and inject tokens at runtime.

### .NET Framework / Web Forms (HttpClient)

```csharp
public async Task SaveItemAsync(string id, string description, string token)
{
   var url = "https://api.businesscentral.dynamics.com/v2.0/{{tenantId}}/{{environment}}/api/v2.0/companies({{companyId}})/{{entitySetName}}/Microsoft.NAV.SaveItem";
   using var client = new HttpClient();
   client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
   var payload = new { id = id, description = description, blocked = false };
   var content = new StringContent(JsonConvert.SerializeObject(payload), Encoding.UTF8, "application/json");
   var resp = await client.PostAsync(url, content);
   resp.EnsureSuccessStatusCode();
}
```

### ASP.NET Core / .NET 6+

```csharp
public async Task<List<ItemDto>> GetItemsAsync(HttpClient client, string token)
{
   var url = "https://api.businesscentral.dynamics.com/v2.0/{{tenantId}}/{{environment}}/api/v2.0/companies({{companyId}})/{{entitySetName}}/Microsoft.NAV.GetItems";
   client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
   var resp = await client.PostAsync(url, new StringContent("{}", Encoding.UTF8, "application/json"));
   resp.EnsureSuccessStatusCode();
   var json = await resp.Content.ReadAsStringAsync();
   return JsonSerializer.Deserialize<List<ItemDto>>(json)!;
}
```

### Simple GET (any .NET)

```csharp
public async Task<string> GetItemAsync(HttpClient client, string id, string token)
{
   var url = "https://api.businesscentral.dynamics.com/v2.0/{{tenantId}}/{{environment}}/api/v2.0/companies({{companyId}})/{{entitySetName}}?$filter=id eq '" + id + "'";
   client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
   var resp = await client.GetAsync(url);
   resp.EnsureSuccessStatusCode();
   return await resp.Content.ReadAsStringAsync();
}
```

### JavaScript (fetch)

```javascript
async function deleteItem(id, token) {
  const url = `https://api.businesscentral.dynamics.com/v2.0/{{tenantId}}/{{environment}}/api/v2.0/companies({{companyId}})/{{entitySetName}}/Microsoft.NAV.DeleteItem`;
  const res = await fetch(url, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
    },
    body: "{}",
  });
  if (!res.ok) throw new Error(await res.text());
}
```

### jQuery

```javascript
function saveItem(id, description, token) {
  const url = `https://api.businesscentral.dynamics.com/v2.0/{{tenantId}}/{{environment}}/api/v2.0/companies({{companyId}})/{{entitySetName}}/Microsoft.NAV.SaveItem`;
  return $.ajax({
    url,
    method: "POST",
    contentType: "application/json",
    headers: { Authorization: `Bearer ${token}` },
    data: JSON.stringify({ id, description, blocked: false }),
  });
}
```

Notes:

- Keep tokens and secrets out of source control.
- Use company **ID** (GUID) or URL-encoded name consistently across calls.
- Replace `{{entitySetName}}` with the `EntitySetName` defined on your API page.

## Pitfalls

- **Missing `SourceTable` on API page**: AL compile-time error (page cannot build). Ensure `PageType = API` pages define `SourceTable` (e.g., a lightweight logging table).
- **Forgot `SourceTableTemporary = true`**: The dummy record inserted on open will write to the real table, pollute logs, and can cause key collisions later. Mark the SourceTable as temporary on the API page.
- **Empty entity set (no dummy record)**: Some OData clients and discovery flows may fail to list unbound actions or return `404 Not Found` when posting to `.../{{entitySetName}}/Microsoft.NAV.<Action>` if the entity set is empty. Keep a harmless dummy row to ensure the entity set is non-empty:

```al
trigger OnOpenPage()
begin
   // Insert a dummy record so the entity set is not empty
   if not Rec.FindFirst() then begin
      Rec."Entry No." := 1;
      Rec.Insert();
   end;
end;
```

- **Wrong or missing OData key**: If `ODataKeyFields` is not set correctly, some tooling may not be able to address records or build links. Use a stable integer key like `"Entry No."` or accept the default `SystemId` behavior.
- **Date/time payloads**: Sending non-ISO dates can cause `400 Bad Request` parsing errors. Use `YYYY-MM-DDTHH:mm:ssZ` or include the offset (e.g., `+03:00`).
