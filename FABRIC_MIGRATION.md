# NCT Feasibility — Fabric Warehouse Migration Guide

This document covers everything needed to migrate the feasibility tool from Supabase to Microsoft Fabric Warehouse. It is a preparation document — no Fabric subscription is required to read it. Implement it once NCT has Fabric and an Entra ID / Microsoft 365 tenant.

---

## 1. What Changes

| Concern | Supabase (current) | Fabric Warehouse (target) |
|---|---|---|
| **Database** | PostgreSQL (Supabase cloud) | Fabric Warehouse (T-SQL / DW) |
| **Auth** | Supabase anon key (public) | MSAL.js PKCE → Azure AD token |
| **API** | Supabase REST (`/rest/v1/`) | Fabric REST execute-query endpoint |
| **Frontend** | `sbFetch()` helper | `fabricFetch()` replacement |
| **Schema** | 4 PostgreSQL tables | Same 4 tables, T-SQL syntax |
| **Pilot phase** | No RLS (internal only) | Row-level security optional via Fabric workspace roles |

Nothing in the HTML structure, input matrix, or export logic changes. Only the persistence layer (save/load) and auth method change.

---

## 2. Prerequisites

Before you start:

1. **Microsoft Fabric** capacity purchased and provisioned
2. **Microsoft 365** tenant (your Azure AD / Entra ID lives here)
3. **Azure portal** access — you need permission to register an app in Entra ID
4. **Fabric Workspace** created, with a **Warehouse** item inside it

---

## 3. Step 1 — Azure AD App Registration

This gives the browser tool an identity it can use to obtain an access token.

### 3.1 Register the app

1. Go to **[Azure Portal](https://portal.azure.com) → Azure Active Directory → App registrations → New registration**
2. Fill in:
   - **Name:** `NCT Feasibility Tool`
   - **Supported account types:** `Accounts in this organizational directory only`
   - **Redirect URI:** `Single-page application (SPA)` → enter the URL where the tool is hosted, e.g.:
     - `https://kimhoo-ong.github.io/NCT-feasi/NCT_Feasi_Input.html` (GitHub Pages)
     - Or `http://localhost:3000` for local testing
3. Click **Register**
4. Note the **Application (client) ID** — you will paste this into the HTML as `MSAL_CLIENT_ID`
5. Note the **Directory (tenant) ID** — you will paste this as `MSAL_TENANT_ID`

### 3.2 Set API permissions

In the registered app → **API permissions → Add a permission**:

1. Select **APIs my organization uses** → search for **Power BI Service**
2. Add delegated permission: **`Dataset.ReadWrite.All`** (covers Fabric REST calls in preview)
3. Also add: **`Workspace.Read.All`**
4. Click **Grant admin consent** (requires tenant admin)

> **Note:** Fabric REST API permissions are still being consolidated under the Power BI Service umbrella as of mid-2025. If you see a dedicated "Microsoft Fabric" entry, use that instead and select `Warehouse.ReadWrite.All`.

### 3.3 Enable public client flows

In the registered app → **Authentication → Advanced settings**:
- Set **Allow public client flows** → **Yes** (enables PKCE without a client secret)

---

## 4. Step 2 — Find Your Fabric IDs

You need two identifiers for the REST API.

### 4.1 Workspace ID

1. Open [Fabric portal](https://fabric.microsoft.com)
2. Navigate to your workspace
3. The URL contains the workspace ID: `https://fabric.microsoft.com/groups/{WORKSPACE_ID}/...`
4. Copy the GUID after `/groups/`

### 4.2 Warehouse ID

1. Inside the workspace, click on your Warehouse item
2. URL contains: `https://fabric.microsoft.com/groups/{WORKSPACE_ID}/warehouses/{WAREHOUSE_ID}`
3. Copy the GUID after `/warehouses/`

---

## 5. Step 3 — Create the Schema (T-SQL DDL)

Run these statements in Fabric Warehouse (Query editor or SSMS connected to the warehouse endpoint).

```sql
-- ============================================================
-- NCT Feasibility — Fabric Warehouse Schema
-- Run once. Drop tables first if recreating from scratch.
-- ============================================================

-- NOTE: Fabric Warehouse does not support PRIMARY KEY, FOREIGN KEY,
-- DEFAULT, UNIQUE, or CHECK constraints in CREATE TABLE.
-- Columns only. id is supplied by the app (crypto.randomUUID());
-- created_at is supplied via SYSDATETIME() in the INSERT statement.

CREATE TABLE dbo.snapshots (
    id             UNIQUEIDENTIFIER,
    project_name   NVARCHAR(255),
    project_code   NVARCHAR(100),
    stage          NVARCHAR(10),
    scenario       NVARCHAR(50),
    status         NVARCHAR(50),
    version_date   DATE,
    wacc           DECIMAL(6,4),
    created_at     DATETIME2
);

CREATE TABLE dbo.hierarchy (
    snapshot_id    UNIQUEIDENTIFIER,
    l1_company     NVARCHAR(255),
    l2_plot_code   NVARCHAR(100),
    l2_plot_name   NVARCHAR(255),
    l3_phase       NVARCHAR(255),
    l4_subphase    NVARCHAR(255),
    l3_sort_order  INT,
    l4_sort_order  INT
);

CREATE TABLE dbo.sales (
    snapshot_id      UNIQUEIDENTIFIER,
    l3_phase         NVARCHAR(255),
    l4_subphase      NVARCHAR(255),
    l5_component     NVARCHAR(255),
    product_type     NVARCHAR(100),
    units            INT,
    nfa_per_unit     DECIMAL(12,2),
    asp_psf          DECIMAL(12,2),
    bumi_pct         DECIMAL(5,2),
    cross_subsidy_amt DECIMAL(18,2),
    carpark_revenue  DECIMAL(18,2),
    sort_order       INT
);

CREATE TABLE dbo.cost (
    snapshot_id    UNIQUEIDENTIFIER,
    l3_phase       NVARCHAR(255),
    l4_subphase    NVARCHAR(255),
    l5_component   NVARCHAR(255),
    pillar_code    NVARCHAR(20),
    item_code      NVARCHAR(100),
    description    NVARCHAR(500),
    budget_amt     DECIMAL(18,2),
    sort_order     INT
);
GO
```

> **Note on `REFERENCES`:** Fabric Warehouse as of mid-2025 parses but does not enforce foreign key constraints. They serve as documentation only. Remove them if Fabric gives an error.

---

## 6. Step 4 — Replace Authentication in the HTML

### 6.1 Add MSAL.js to the HTML `<head>`

Replace the Supabase CDN import with MSAL:

```html
<!-- Remove: -->
<!-- <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/..."></script> -->

<!-- Add: -->
<script src="https://alcdn.msauth.net/browser/2.38.3/js/msal-browser.min.js"></script>
```

### 6.2 Configuration constants (replace Supabase constants)

```js
// ── Fabric / Azure AD config ─────────────────────────────
const MSAL_CLIENT_ID   = 'PASTE_YOUR_APP_CLIENT_ID_HERE';
const MSAL_TENANT_ID   = 'PASTE_YOUR_TENANT_ID_HERE';
const FABRIC_WORKSPACE = 'PASTE_YOUR_WORKSPACE_GUID_HERE';
const FABRIC_WAREHOUSE = 'PASTE_YOUR_WAREHOUSE_GUID_HERE';

// Fabric SQL execute endpoint
const FABRIC_ENDPOINT =
  `https://api.fabric.microsoft.com/v1/workspaces/${FABRIC_WORKSPACE}/warehouses/${FABRIC_WAREHOUSE}/queryexecution`;

// MSAL instance
const msalApp = new msal.PublicClientApplication({
  auth: {
    clientId: MSAL_CLIENT_ID,
    authority: `https://login.microsoftonline.com/${MSAL_TENANT_ID}`,
    redirectUri: window.location.origin + window.location.pathname,
  },
  cache: { cacheLocation: 'sessionStorage' },
});

// Scopes — Power BI / Fabric delegated
const FABRIC_SCOPES = ['https://analysis.windows.net/powerbi/api/.default'];
```

### 6.3 Auth helper — get a token silently, prompt if needed

```js
async function getAccessToken() {
  const accounts = msalApp.getAllAccounts();
  const request  = { scopes: FABRIC_SCOPES, account: accounts[0] };
  try {
    // Silent first (uses cached token if still valid)
    const result = await msalApp.acquireTokenSilent(request);
    return result.accessToken;
  } catch (e) {
    if (e instanceof msal.InteractionRequiredAuthError) {
      // Popup login if silent fails (first time or token expired)
      const result = await msalApp.acquireTokenPopup(request);
      return result.accessToken;
    }
    throw e;
  }
}
```

### 6.4 Fabric fetch helper — replaces `sbFetch()`

```js
// Executes a T-SQL statement against Fabric Warehouse.
// Returns parsed JSON rows (array of objects).
async function fabricFetch(sql, bindValues = []) {
  const token = await getAccessToken();
  const body  = {
    statements: [{ statement: sql, parameters: bindValues }]
  };
  const res = await fetch(FABRIC_ENDPOINT, {
    method:  'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type':  'application/json',
    },
    body: JSON.stringify(body),
  });
  if (!res.ok) {
    const err = await res.text();
    throw new Error(`Fabric API ${res.status}: ${err}`);
  }
  const data = await res.json();
  // The Fabric execute API returns results in data.results[0].data
  const rows = data?.results?.[0]?.data ?? [];
  return rows;
}
```

> **Note:** The Fabric Warehouse query execution REST API shape may change as the feature graduates from preview. Check the [Fabric REST API docs](https://learn.microsoft.com/en-us/rest/api/fabric/warehouse) for the current request/response schema.

---

## 7. Step 5 — Rewrite Save and Load Logic

### 7.1 Save — INSERT pattern

The current `buildSavePayload()` and Supabase batch POST become parameterized T-SQL INSERTs.

```js
async function saveToFabric() {
  const payload = buildSavePayload();   // existing function, unchanged

  // 1. Insert snapshot, capture generated ID
  const snapSql = `
    INSERT INTO dbo.snapshots
      (project_name, project_code, l1_company, l2_plot_code, l2_plot_name,
       location, stage, scenario, status, version_date, wacc)
    OUTPUT INSERTED.id
    VALUES (?,?,?,?,?,?,?,?,?,?,?)`;

  const snapRow = await fabricFetch(snapSql, [
    payload.project_name, payload.project_code, payload.l1_company,
    payload.l2_plot_code, payload.l2_plot_name, payload.location,
    payload.stage, payload.scenario, payload.status,
    payload.version_date, payload.wacc,
  ]);
  const snapId = snapRow[0]?.id;

  // 2. Insert hierarchy rows
  for (const h of payload.hierarchy) {
    await fabricFetch(
      `INSERT INTO dbo.hierarchy (snapshot_id,l3_phase,l4_subphase,l3_sort_order,l4_sort_order)
       VALUES (?,?,?,?,?)`,
      [snapId, h.l3_phase, h.l4_subphase, h.l3_sort_order, h.l4_sort_order]
    );
  }

  // 3. Insert sales rows
  for (const s of payload.sales) {
    await fabricFetch(
      `INSERT INTO dbo.sales
         (snapshot_id,l3_phase,l4_subphase,l5_component,product_type,
          units,nfa_per_unit,asp_psf,bumi_pct,cross_subsidy_amt,carpark_revenue,sort_order)
       VALUES (?,?,?,?,?,?,?,?,?,?,?,?)`,
      [snapId, s.l3_phase, s.l4_subphase, s.l5_component, s.product_type,
       s.units, s.nfa_per_unit, s.asp_psf, s.bumi_pct,
       s.cross_subsidy_amt, s.carpark_revenue, s.sort_order]
    );
  }

  // 4. Insert cost rows
  for (const c of payload.cost) {
    await fabricFetch(
      `INSERT INTO dbo.cost
         (snapshot_id,l3_phase,l4_subphase,l5_component,
          pillar_code,item_code,description,budget_amt,sort_order)
       VALUES (?,?,?,?,?,?,?,?,?)`,
      [snapId, c.l3_phase, c.l4_subphase, c.l5_component,
       c.pillar_code, c.item_code, c.description, c.budget_amt, c.sort_order]
    );
  }

  alert('Saved to Fabric Warehouse ✓');
}
```

### 7.2 Load — SELECT pattern

```js
async function listSnapshotsFabric() {
  const rows = await fabricFetch(
    `SELECT id, project_name, project_code, stage, status, version_date, created_at
     FROM dbo.snapshots
     ORDER BY created_at DESC`
  );
  return rows;   // render in Load modal the same way as before
}

async function loadSnapshotFabric(snapId) {
  const [meta, hierarchy, sales, costs] = await Promise.all([
    fabricFetch(`SELECT * FROM dbo.snapshots WHERE id = ?`, [snapId]),
    fabricFetch(`SELECT * FROM dbo.hierarchy WHERE snapshot_id = ? ORDER BY l3_sort_order, l4_sort_order`, [snapId]),
    fabricFetch(`SELECT * FROM dbo.sales    WHERE snapshot_id = ? ORDER BY sort_order`, [snapId]),
    fabricFetch(`SELECT * FROM dbo.cost     WHERE snapshot_id = ? ORDER BY sort_order`, [snapId]),
  ]);
  restoreFromData(meta[0], hierarchy, sales, costs);  // existing restoreFromSupabase() renamed
}
```

---

## 8. Step 6 — Login Button

Because Fabric requires a signed-in Azure AD account, add a login button to the toolbar:

```html
<button id="login-btn" onclick="signIn()" style="display:none">🔑 Sign In</button>
<span id="user-label" style="font-size:11px;color:#aaa"></span>
```

```js
async function signIn() {
  const result = await msalApp.loginPopup({ scopes: FABRIC_SCOPES });
  document.getElementById('user-label').textContent = result.account.username;
  document.getElementById('login-btn').style.display = 'none';
}

// On page load — check if already signed in
msalApp.initialize().then(() => {
  const accounts = msalApp.getAllAccounts();
  if (accounts.length) {
    document.getElementById('user-label').textContent = accounts[0].username;
  } else {
    document.getElementById('login-btn').style.display = '';
  }
});
```

---

## 9. Migration Checklist

- [ ] Fabric capacity purchased and workspace created
- [ ] Warehouse item created inside workspace
- [ ] Azure AD app registered; client ID and tenant ID noted
- [ ] API permissions granted (Power BI delegated) + admin consent given
- [ ] Public client / PKCE flows enabled
- [ ] Redirect URI added to app registration (GitHub Pages URL + localhost)
- [ ] T-SQL DDL run in Fabric Warehouse (4 tables + indexes)
- [ ] `MSAL_CLIENT_ID`, `MSAL_TENANT_ID`, `FABRIC_WORKSPACE`, `FABRIC_WAREHOUSE` pasted into HTML
- [ ] `sbFetch()` calls replaced with `fabricFetch()`
- [ ] Save / Load functions updated (see §7)
- [ ] Login button added and `msalApp.initialize()` called on page load
- [ ] Tested locally with `http://localhost:3000` redirect URI
- [ ] Tested on GitHub Pages with production redirect URI
- [ ] Historical Supabase snapshots optionally exported and imported into Fabric

---

## 10. Supabase Data Export (Optional)

If you want to carry over existing snapshots from Supabase:

1. In Supabase dashboard → **Table Editor** → export each table to CSV
2. In Fabric Warehouse → **Data pipelines** or **COPY INTO** to bulk-load the CSVs
3. UUIDs from Supabase (`gen_random_uuid()`) are compatible with `UNIQUEIDENTIFIER` in T-SQL

---

## 11. References

- [Microsoft Fabric REST API — Warehouse](https://learn.microsoft.com/en-us/rest/api/fabric/warehouse)
- [MSAL.js browser library](https://github.com/AzureAD/microsoft-authentication-library-for-js)
- [Azure AD app registration quickstart](https://learn.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)
- [Fabric Warehouse T-SQL support](https://learn.microsoft.com/en-us/fabric/data-warehouse/tsql-surface-area)
