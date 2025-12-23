# Business Central Online Integration Guide

This guide allows developers to quickly adapt portal integrations from BC On-Premises to BC Online.

## 1. Quick Start Checklist
- [ ] **Azure AD App**: Register app, get Client ID/Secret, grant `Dynamics 365 Business Central > API.ReadWrite.All`.
- [ ] **Postman**: Import collection, set environment variables (`tenantId`, `clientId`, `clientSecret`, `companyId`).
- [ ] **Connect**: Get Bearer Token -> Call API Endpoint.

---

## 2. Authentication (OAuth 2.0)
BC Online uses Azure AD (Entra ID) for authentication. Basic Auth is deprecated.

### A. Azure AD App Setup
1.  **Portal**: Azure Portal > App registrations > New.
2.  **Secret**: Certificates & secrets > New client secret (Copy Value!).
3.  **Permissions**: API permissions > Add > Dynamics 365 Business Central > Application permissions > `API.ReadWrite.All` > **Grant Admin Consent**.

### B. Get Bearer Token
**Endpoint**: `https://login.microsoftonline.com/{{tenantId}}/oauth2/v2.0/token`
**Method**: `POST` (Body: `x-www-form-urlencoded`)

| Key | Value |
| :--- | :--- |
| `grant_type` | `client_credentials` |
| `client_id` | `{{clientId}}` |
| `client_secret` | `{{clientSecret}}` |
| `scope` | `https://api.businesscentral.dynamics.com/.default` |

> **Tip**: Cache the `access_token` and reuse it until it expires (default: 1 hour). Do not request a new token for every API call.

---

## 3. API Endpoints
Base URL format:
`https://api.businesscentral.dynamics.com/v2.0/{{tenantId}}/{{environment}}/api/{{publisher}}/{{group}}/{{version}}/companies({{companyId}})/{{entitySetName}}`

### Standard vs. Custom
- **Standard APIs**: `.../api/v2.0/companies({{companyId}})/customers`
- **Custom APIs**: Defined by your implementation (see Section 4).

> **Important**: Always use `companyId` (GUID) from the `/companies` endpoint. Using names is fragile with special characters.

---

## 4. Developing Custom APIs (AL)
Expose logic via **API Pages** backed by **ServiceEnabled Procedures**.

### A. API Page Template
Create a Page with `PageType = API`. Use a **temporary source table** to handle unbound actions cleanly.

```al
page 50000 "Portal Operations"
{
    PageType = API;
    APIPublisher = 'mycompany';
    APIGroup = 'portal';
    APIVersion = 'v1.0';
    EntityName = 'operation';
    EntitySetName = 'operations';
    SourceTable = "Portal Buffer"; // Temporary table
    SourceTableTemporary = true;
    ODataKeyFields = "Entry No.";
    
    // Ensure entity set is not empty for OData discovery
    trigger OnOpenPage()
    begin
        if not Rec.FindFirst() then begin
            Rec.Init();
            Rec."Entry No." := 1;
            Rec.Insert();
        end;
    end;
}
```

### B. Service Procedure (Unbound Action)
Add `[ServiceEnabled]` methods to handle specific logic (create header, add lines, posts).

```al
[ServiceEnabled]
procedure CreateRequisition(employeeNo: Code[20]; description: Text) result: Text
var
    ReqHeader: Record "Requisition Header";
    Response: JsonObject;
begin
    // 1. Business Logic
    ReqHeader.Init();
    ReqHeader.Insert(true); // Triggers No. Series
    ReqHeader.Validate("Employee No.", employeeNo);
    ReqHeader.Description := description;
    ReqHeader.Modify(true);

    // 2. Return JSON Response
    Response.Add('success', true);
    Response.Add('documentNo', ReqHeader."No.");
    Response.WriteTo(result);
end;
```

### C. Calling the Action (POST)
**URL**: `.../operations/Microsoft.NAV.CreateRequisition`
**Body (JSON)**:
```json
{
    "employeeNo": "EMP001",
    "description": "Office Supplies"
}
```

---

## 5. Implementation Examples

### C# (HttpClient)
```csharp
public async Task<string> CallBcAction(string token, string actionName, object payload)
{
    var url = $"https://api.businesscentral.dynamics.com/v2.0/{{tenantId}}/{{env}}/api/mycompany/portal/v1.0/companies({{companyId}})/operations/Microsoft.NAV.{actionName}";
    
    using var client = new HttpClient();
    client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
    
    var jsonPayload = JsonConvert.SerializeObject(payload);
    var content = new StringContent(jsonPayload, Encoding.UTF8, "application/json");
    
    var response = await client.PostAsync(url, content);
    response.EnsureSuccessStatusCode();
    
    return await response.Content.ReadAsStringAsync();
}
```

### JavaScript (Fetch)
```javascript
const callBcAction = async (token, actionName, payload) => {
    const url = `https://api.businesscentral.dynamics.com/v2.0/{{tenantId}}/{{env}}/api/mycompany/portal/v1.0/companies({{companyId}})/operations/Microsoft.NAV.{actionName}`;
    
    const response = await fetch(url, {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(payload)
    });

    if (!response.ok) throw new Error(await response.text());
    return await response.json();
};
```

---

## 6. Common Pitfalls & Fixes
| Issue | Solution |
| :--- | :--- |
| **401 Unauthorized** | Check permissions (`API.ReadWrite.All`) and ensure **Admin Consent** is granted in Azure AD. |
| **404 Not Found** | Verify `Publisher`, `Group`, `Version`, and `EntitySetName` in URL. Ensure Extension is published to the correct environment. |
| **400 Bad Request** | Check JSON body format. For dates, use ISO 8601 (`YYYY-MM-DDTHH:mm:ssZ`). |
| **Action not found** | Ensure the API Page has a record (dummy row) visible. OData won't show actions if the collection is empty. |
