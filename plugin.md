---
title: Plugin (Blank Menu)
section: Customization
order: 14
---

# Plugin (Blank Menu)

A **Plugin** is a [Module Editor](module_editor.md) menu whose screen is
**not** built inside Alaya. Instead, Alaya opens a web page that **you
host** inside an `iframe`, hands it the current user's context, and lets
it call the [Alaya REST API](api_access.md) on behalf of that user. To the
end-user it looks like any other Alaya screen: it opens in a tab, sits in
the navigation under "My Module", and respects the same user-group
permissions.

In the Module Editor this menu type is called **Plugin**. Internally it
is the **Blank** menu type (`UT_MenuType = 1`).

This page explains how a Plugin works at runtime and walks through
building one from end to end.

## When to use a Plugin

| Use a **Master Data** menu when... | Use a **Plugin** menu when... |
|----|----|
| A simple form with the standard field types is enough. | You need a custom UI (dashboards, wizards, calendars, maps, third-party widgets). |
| You want Alaya to render the list and detail screens for you. | You already have, or want to build, your own web application. |
| No code should be written or hosted. | You are able to host an HTTPS web page and call REST APIs. |

The two can be combined. A common pattern is one module that contains a
Plugin menu (your custom UI) plus one or more Master Data menus that act
as the data store for it, read and written through the `Udf` API
(see [Reading and writing Alaya data](#reading-and-writing-alaya-data)).

## How a Plugin works

```
┌────────────────────────── Alaya (menu tab) ─────────────────────────────┐
│  Toolbar (optional): Back | Save | Copy                                 │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │  <iframe src="https://your-host/plugin?sid=…&v=l">                  │ │
│ │                                                                     │ │
│ │     Your page                                                       │ │
│ │       ▲ 1. ALAYA_INIT  (user, company, nonce, apiurl, menuInfo …)   │ │
│ │       │ 2. ALAYA_READY (reply within 10 seconds)                    │ │
│ │       │ 3. ALAYA_TOOLBAR / ALAYA_TOOLBAR_VISIBLE / ALAYA_TAB        │ │
│ └───────┼─────────────────────────────────────────────────────────────┘ │
└─────────┼───────────────────────────────────────────────────────────────┘
          │ 4. nonce + DevKey  →  POST {apiurl}token  →  access token
          ▼ 5. Authorization: Bearer …  →  {apiurl}api/…
   Alaya REST API
```

1. A user clicks the Plugin menu. Alaya checks that the client's licence
   includes the **Integration API addon (BSMA13)** and that the menu has
   an **External URL**.
2. Alaya loads your **External URL** in an `iframe`, appending `sid` and
   `v` query parameters (see [Query string](#query-string)).
3. When the iframe finishes loading, Alaya sends your page an
   **`ALAYA_INIT`** message with the user, company, language, API URL, a
   short-lived **nonce**, and the menu's access rights.
4. Your page must answer with **`ALAYA_READY`** within **10 seconds**.
   Otherwise Alaya replaces the screen with a timeout error.
5. To call the Alaya API, exchange the nonce (together with your
   **DevKey**) for an access token, then call endpoints as the logged-in
   user.

All communication between Alaya and your page uses the browser
`window.postMessage` API.

### List view and detail view

A Plugin menu can be opened in two modes. Alaya tells your page which one
through `viewType` in `ALAYA_INIT` (and the `v` query parameter):

| View | `viewType` | `v` | Opened by |
|----|----|----|----|
| List | `LIST` | `l` | The user clicks the menu in the navigation. |
| Detail | `DETAIL` | `d` | A detail URL produced by `api/Udf/ZZTGetPageUrl` for this menu is opened, for example via [`ALAYA_TAB`](#message-reference). |

The system toolbar's **Back** button is only shown in the `DETAIL` view.

```
Note: ALAYA_INIT does not carry a record key. If your detail view needs to know which record to show, keep that state in your own application.
```

## Message reference

Every message is an object of the form `{ type: "...", data: { ... } }`.

### `ALAYA_INIT` (Alaya → Plugin)

Sent once, when the iframe has loaded. **Register your `message`
listener while the page is first executing** (for example, in an inline
script in `<head>`), not after asynchronous work, otherwise you can miss
it.

```json
{
  "type": "ALAYA_INIT",
  "data": {
    "targetApp": "ALAYA_UDF",
    "clientID": "1",
    "companyID": "1",
    "userID": "12",
    "adminYN": "false",
    "language": "en",
    "client": "CLIENT01",
    "username": "jsmith",
    "apiurl": "https://api.example.com/v3/",
    "nonce": "…",
    "viewType": "LIST",
    "menuInfo": {
      "AppId": 5,
      "AppName": "FranchiseManagement",
      "AppCaption": "Franchise Management",
      "UT_ID": 31,
      "UT_TableName": "FranchiseDashboard",
      "UT_Caption": "Dashboard",
      "ListAccess":   { "VIEW": true, "ADD": true, "DELETE": false, "COPY": true },
      "DetailAccess": { "VIEW": true, "SAVE / EDIT": true, "DELETE": false, "COPY": true }
    }
  }
}
```

| Field | Description |
|----|----|
| `targetApp` | Always `ALAYA_UDF`. |
| `client` | The client code (used as `clientid` when requesting a token). |
| `clientID` | Internal numeric client identifier. |
| `companyID` | The company the user is currently working in. Pass it to company-scoped APIs such as `ZZTGetDataTable`. |
| `username` | The logged-in user's ID. Used as `username` when requesting a token. |
| `userID` | The logged-in user's numeric ID. |
| `adminYN` | `"true"` / `"false"`. Whether the user is an administrator. |
| `language` | The user's Alaya display language. |
| `apiurl` | Base URL of the Alaya API for this client. Already resolved for you, no need to call `GetUrl`. |
| `nonce` | Short-lived value used to obtain an API token (see [Getting an API token](#getting-an-api-token)). Valid for **5 minutes**. |
| `viewType` | `LIST` or `DETAIL`. |
| `menuInfo` | Identifies the module (`AppId`, `AppName`) and menu (`UT_ID`, `UT_TableName`), and lists what the current user is allowed to do. See [Access rights](#access-rights). `menuInfo` is `{}` if the menu cannot be resolved. |

### `ALAYA_READY` (Plugin → Alaya)

Reply to `ALAYA_INIT` to tell Alaya your page loaded correctly.

```js
window.parent.postMessage(
  { type: 'ALAYA_READY', data: { status: 'ok' } },
  '*'
);
```

If Alaya does not receive it within 10 seconds, the user sees
*"Timed out after 10 seconds - The external page did not respond in
time."* with a **Retry** button. Alaya also ignores any message that does
not come from the origin of your External URL.

### `ALAYA_TOOLBAR` (Alaya → Plugin)

Sent when the user clicks a button on the Alaya system toolbar (only when
**Use System Toolbar** is enabled for the menu).

```json
{ "type": "ALAYA_TOOLBAR", "data": { "targetApp": "ALAYA_UDF", "name": "Save" } }
```

`name` is one of `Back`, `Save` or `Copy`. Alaya only forwards the click;
performing the action (saving, navigating back, copying) is up to your
page.

### `ALAYA_TOOLBAR_VISIBLE` (Plugin → Alaya)

Show or hide a system toolbar button at runtime.

```js
window.parent.postMessage(
  { type: 'ALAYA_TOOLBAR_VISIBLE', data: { name: 'Save', value: true } },
  '*'
);
```

Initial visibility when the toolbar is enabled:

| Button | Initially visible when |
|----|----|
| **Back** | `viewType` is `DETAIL`. |
| **Save** | The user has the *Edit* access right on this menu. |
| **Copy** | The user has the *Clone* access right on this menu. |

If none of the buttons are visible, the toolbar collapses and the iframe
uses the full height.

### `ALAYA_TAB` (Plugin → Alaya)

Ask Alaya to open a new tab.

```js
window.parent.postMessage(
  { type: 'ALAYA_TAB', data: { tabName: 'Order (000123)', tabUrl: '/modules/customization/u_UdfDisplayDetail.aspx?EncData=…' } },
  '*'
);
```

`tabUrl` must be an **Alaya-generated relative URL**, typically obtained
from `api/Udf/ZZTGetPageUrl`. URLs starting with `http` are rejected. To
show a different external page, create another Plugin menu with its own
External URL.

### Query string

Alaya appends these parameters to your External URL (in addition to any
you already have in it). They are for Alaya's own use, and your page
should read the same information from `ALAYA_INIT` instead.

| Parameter | Meaning |
|----|----|
| `sid` | Internal session key. Do not use. |
| `v` | `l` (list) or `d` (detail). Equivalent to `viewType`. |
| `EncData` | Encrypted Alaya state. Opaque to your page. |

## Access rights

There are two layers, and they are independent:

1. **Who can open the menu**: controlled in Alaya under
   **Staff** → **User Group**, exactly like any other Alaya menu. In the
   Plugin View Editor, the **View** option controls whether Alaya
   enforces the *View* right when the page is opened; if it is ticked
   and the user does not have it, Alaya shows its *Unauthorized* page
   instead of loading your URL.
2. **What the user can do inside your page**: `menuInfo.ListAccess` and
   `menuInfo.DetailAccess` in `ALAYA_INIT` list each action and whether
   the current user (in the current company) has it. The actions listed
   are those ticked in the Plugin View Editor (`DetailAccess`) and the
   standard list actions (`ListAccess`).

```
Note: menuInfo flags are there to drive your UI (hide a Delete button, disable Save). They run in the browser, so always validate permissions again on your own backend before changing data.
```

## Getting an API token

Your page must not ask users for their Alaya password. Instead, it
exchanges the `nonce` from `ALAYA_INIT` together with your **DevKey**
for an access token that represents the logged-in user.

**Request**

    POST {apiurl}token
    Content-Type: application/x-www-form-urlencoded

    grant_type=password&clientid={client}&username={username}&devkey={DevKey}&nonce={nonce}

| Parameter | Value |
|----|----|
| `grant_type` | `password` |
| `clientid` | `client` from `ALAYA_INIT` |
| `username` | `username` from `ALAYA_INIT` |
| `devkey` | Your DevKey (see [Step 1](#building-a-plugin-step-by-step)) |
| `nonce` | `nonce` from `ALAYA_INIT`. Case-sensitive, valid for 5 minutes. |

**Response**: `token_type`, `access_token`, and normally a
`refresh_token`. Use them on every API call:

    Authorization: {token_type} {access_token}

**Refreshing**: after the nonce has expired, renew the token without a
new nonce:

    POST {apiurl}token
    grant_type=refresh_token&refresh_token={refresh_token}&clientid={client}&devkey={DevKey}

A fresh nonce is issued every time Alaya loads your page.

```
Important: your DevKey is a secret. Do not hard-code it in JavaScript that is served to the browser, and do not commit it to a public repository. In production, send the nonce, client and username from your page to your own backend; let the backend hold the DevKey, call {apiurl}token, and either return a short-lived token to the page or call the Alaya API itself. The bundled sample calls the token endpoint straight from the browser only for demonstration.
```

## Reading and writing Alaya data

Once you have a token, your page can use the standard Alaya API. The
`Udf` controller is the one relevant to Module Editor data. All
endpoints are under `{apiurl}api/Udf/` and return the usual
`{ "Result": true, "Message": "", "Obj": … }` envelope. See
[API Access](api_access.md) for the full reference.

| Endpoint | Method | Purpose |
|----|----|----|
| `GetUdfAppUIById?id={AppId}` | GET | Get the module and all its menus (`Obj.UdfTables`). Use `menuInfo.AppId` for the id. |
| `GetUdfTableByCode?code={UT_TableName}` | GET | Get one menu's definition. |
| `GetUdfColumnList?tableName={UT_TableName}` | GET | List the fields (columns) of a menu. |
| `ZZTGetDataTable` | POST | List rows of a Master Data menu. Body: `{ "tableName", "companyId", "searchKeyword", "lstColumn" }`. Returns up to 500 rows unless a search keyword is supplied. |
| `ZZTGetRowById` | POST | Get one row. Body: `{ "tableName", "primaryKey" }`. |
| `ZZTSaveRow` | POST | Insert (`primaryKey: 0`) or update a row. Body: `{ "tableName", "primaryKey", "columnValuePairs": { "FieldName": value } }`. Returns the row's primary key. |
| `ZZTDeleteRow` | POST | Delete a row (permanent for Master Data). Body: `{ "tableName", "primaryKey" }`. |
| `ZZTGetPageUrl` | POST | Get Alaya detail-page URLs for rows, to open with [`ALAYA_TAB`](#message-reference). Body: `{ "tableName", "primaryKey": [ids], "companyId" }`. Returns `{ id: url }`. |

Things to know about table data:

- `tableName` is always the **Menu Code** (`UT_TableName`).
- Columns returned by `ZZTGetDataTable` carry the physical prefix
  (`ZZC_ID`, `ZZC_Code`, `ZZC_<FieldName>`). When saving with
  `ZZTSaveRow`, use the plain `UC_FieldName` without the prefix.
- `ZZTGetDataTable` filters rows by `companyId` according to the menu's
  **Associated Group** setting.
- A Plugin menu itself has no fields. Store your data in Master Data
  menus of the same module, or in your own database.

## Building a Plugin, step by step

### Step 1 — Register as a developer

1. Open the **Developer Portal** (**DevPortal**) of Alaya's iDealer site.
   The Plugin View Editor contains a direct link, and the Module Editor's
   **DevKey** field has a **Register as Developer** link.
2. Register, then copy your **DevKey**.

Your DevKey identifies you to the Alaya API (it is what allows your page
to obtain tokens) and protects modules you own from being exported by
other developers.

### Step 2 — Prepare the client environment

The Alaya client you test against needs:

- The **Integration API addon (BSMA13)** in its licence. Without it the
  menu shows *"Your license does not include the Integration API addon
  (BSMA13)."*
- An **Admin** user to configure the Module Editor.

### Step 3 — Create the module

1. Go to **Utility** → **Customization** → **Module Editor**.
2. Click **+ New** and fill in **Module Code**, **Module Name** and
   **Sequence**. The Module Code must be at least 3 characters, start
   with a letter and contain letters and digits only.
3. Enter your **DevKey** (use the validate icon to check it). Once a
   DevKey is set on a module, changing the module later requires
   unlocking it with the same DevKey.
4. Confirm.

### Step 4 — Add a Plugin menu

1. Click **+** on the module to add a menu.
2. Enter **Menu Code** and **Menu Name**.
3. For **Type**, choose **Plugin**.
4. Confirm.

```
Note: Menu Code and Type cannot be changed after the menu is saved. A Plugin menu has no Screen Configuration button, because Alaya does not render its form.
```

If your plugin needs to store data in Alaya, also add **Master Data**
menus (Type: *Master Data*) to the same module and design their fields.
Keep them **Active**; your plugin will access them through the `Udf` API.

### Step 5 — Build and host your page

Start from the sample: in the Plugin View Editor click **Download Plugin
Sample** to get `AlayaPluginSample.html`. It demonstrates the complete
handshake, token exchange, toolbar messages, listing tables, loading
rows and opening Alaya tabs.

The minimum a Plugin page must do:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8" />
<title>My Plugin</title>
<script>
  // Register the listener immediately - ALAYA_INIT is sent once, when the iframe loads.
  var alaya = null;

  window.addEventListener('message', function (e) {
    // Only accept messages from the Alaya window that embeds you.
    if (e.source !== window.parent) return;

    var msg = e.data;
    if (!msg || !msg.type) return;

    if (msg.type === 'ALAYA_INIT') {
      alaya = msg.data;                       // user, company, apiurl, nonce, menuInfo, viewType...
      window.parent.postMessage({ type: 'ALAYA_READY', data: { status: 'ok' } }, '*');
      start();                                // render your UI
    }
    else if (msg.type === 'ALAYA_TOOLBAR') {
      if (msg.data.name === 'Save') saveCurrentForm();
    }
  });

  function start() {
    document.getElementById('hello').textContent =
      'Hello ' + alaya.username + ' (' + alaya.viewType + ' view, company ' + alaya.companyID + ')';
  }
  function saveCurrentForm() { /* your code */ }
</script>
</head>
<body>
  <h1 id="hello">Waiting for Alaya…</h1>
</body>
</html>
```

Hosting requirements:

- Served over **HTTPS** with a valid certificate.
- **Allow framing by Alaya.** Do not send `X-Frame-Options: DENY` /
  `SAMEORIGIN`. If you use a `Content-Security-Policy`, set
  `frame-ancestors` to include your client's Alaya site origin.
- **Do not redirect across origins.** Alaya sends messages to the origin
  of the External URL it loaded. If your page redirects the iframe to a
  different origin (for example a login page on another domain), the
  messages will no longer reach it.
- Answer `ALAYA_INIT` with `ALAYA_READY` within 10 seconds of the tab
  opening, so keep your initial page light and do heavy work afterwards.

### Step 6 — Configure the Plugin menu

1. In the Module Editor, click the menu's **Details** button. The
   **Plugin View Editor** opens.
2. **External URL**: enter the `https://` address of your page. This is
   required, and must start with `https://`.
3. **Use System Toolbar** (optional): tick it if you want Alaya's
   Back / Save / Copy buttons above your page. Your page receives their
   clicks as `ALAYA_TOOLBAR`.
4. In the **View** tab on the left, tick the actions that apply to this
   menu: **View**, **Save / Edit**, **Delete**, **Copy**. A new menu
   starts with all four ticked. These become the keys of
   `menuInfo.DetailAccess`.
5. Read the **Terms & Conditions**, then tick the agreement box. You must
   accept again each time a new Plugin menu is created or its URL is
   changed.
6. Click **Save**.

### Step 7 — Test before you release

Both test buttons open your page in a new Alaya tab marked **(Test)**,
using whatever is currently in the **External URL** box, so you can test
a URL before saving it.

| Button | What it does |
|----|----|
| **Test Load** | Opens the page as you, in your current company. |
| **Test View As** | Opens the page as a chosen **user** and **company**. `ALAYA_INIT` then carries that user's `username`, `userID`, `adminYN` and the `menuInfo` access flags computed for them. Use it to verify that your page behaves correctly for users with limited rights without switching accounts. |

The **Use System Toolbar** checkbox is honoured in the test tabs as well.
Open the browser developer tools (console and network tabs) to inspect
the messages and API calls.

### Step 8 — Release to users

1. In the Module Editor, make sure the module and the menu are both
   **Active**.
2. Go to **Staff** → **User Group** and grant the relevant groups access
   to the menu (and the individual actions such as Edit, Delete and
   Clone).
3. Users in those groups will see the menu under "My Module".

### Step 9 — Distribute to other clients (optional)

A module and its menus can be exported to the DevPortal and imported into
other Alaya clients:

- **Export**: use the module's export button, enter your DevKey.
- **Import**: in the Module Editor, choose import, enter your DevKey and
  pick the plugin. **This overwrites an existing module and menus that
  have the same code.**

What is included: the module, its menus, their fields and layouts, and
for each Plugin menu its **External URL** and **Use System Toolbar**
setting. What is **not** included: the DevKey, each menu's User Group
permissions and Screen Configuration, and the View tab actions of the
Plugin View Editor (a newly imported Plugin menu starts with View,
Save / Edit, Delete and Copy ticked; an existing menu keeps its own).

```
After importing into another client, open the Plugin View Editor of each Plugin menu and check the External URL: it is the one that was exported, so change it if the client should use a different host. Then grant access in User Group.
```

## Developing locally

- The External URL must start with `https://` to be **saved**. To
  iterate quickly, use **Test Load** with your local address before
  saving anything.
- If you test against an Alaya site served over HTTPS, serve your local
  page over HTTPS too (for example `https://localhost:5001` with a
  development certificate), because browsers block `http://` iframes
  inside `https://` pages.
- The bundled sample logs every message it sends and receives, which is
  the quickest way to see the real payloads.

## Troubleshooting

| Symptom | Cause and fix |
|----|----|
| *"Your license does not include the Integration API addon (BSMA13)."* | The client's licence has no BSMA13. Contact the Alaya support team to enable it. |
| *"External URL is empty"* | The menu has no External URL. Open the Plugin View Editor, set it and save. |
| *"Timed out after 10 seconds…"* | Your page did not answer `ALAYA_INIT` with `ALAYA_READY`. Check that your listener is registered during the initial page load, that the page is not blocked from framing (`X-Frame-Options` / `frame-ancestors`), that it does not redirect to another origin, and that the URL is reachable. |
| Blank iframe, console shows the page *refused to connect* | Your server forbids being framed. See the hosting requirements in Step 5. |
| `ALAYA_INIT` never arrives | Listener added too late, or the iframe was redirected to a different origin. |
| Token request fails with *invalid_grant* | Nonce expired (5 minutes) or was mistyped (it is case-sensitive), the DevKey is wrong, or the user/client does not match. Use the refresh token, or reload the tab to get a new nonce. |
| *"Unauthorized"* page when opening the menu | The View option is ticked and the user has no View access to the menu. Grant it in **Staff** → **User Group**. |
| `ALAYA_TAB` does nothing | `tabUrl` starts with `http`. Only Alaya-generated relative URLs are accepted. |
| `ALAYA_TOOLBAR_VISIBLE` does nothing | **Use System Toolbar** is off, or `name` is not `Back`, `Save` or `Copy`. |
| DevKey rejected when editing or exporting a module | The module is protected by another DevKey. Use the DevKey that was set on the module. |

## Security checklist

- Serve the page over HTTPS only.
- Check `event.source === window.parent` on every incoming message, and
  where possible check `event.origin` against your client's Alaya site
  origin. Never act on messages from other windows.
- Keep the DevKey on your server. Do not ship it in client-side code or
  commit it to source control.
- Treat `menuInfo` access flags as UI hints only and enforce permissions
  on your backend.
- Collect and store only the Alaya user data you need, and handle it
  according to the applicable data-protection law.
- Do not scrape or bulk-extract data through the API. Usage that degrades
  availability for other clients can be throttled or revoked.

These are also the obligations you accept in the **Terms & Conditions**
shown in the Plugin View Editor (confidentiality, data privacy, security,
API usage, liability, indemnification and termination). Read the
in-product text. It is the binding version.

## Related Pages

- [Module Editor](module_editor.md)
- [User Defined Field](user_defined_field.md)
- [API Access](api_access.md)
