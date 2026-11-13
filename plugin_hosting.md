---
title: Plugin Hosting with an AI Assistant
section: Customization
order: 14
---

# Plugin Hosting with an AI Assistant

A [Plugin](plugin.md) is a web page that you host yourself. You don't
need your own server for it: an AI assistant (Claude, Cursor, VS Code
with Copilot or another MCP client) can write the page and a small
backend for you and publish them on a hosting service such as
**Vercel** or **Cloudflare**. Both services have MCP servers and command
line tools that an assistant can use with your account.

This page explains what to build, how to connect your assistant to the
hosting service, and what to check before you put the page's address
in a Plugin menu. Read [Plugin](plugin.md) first; it describes the
messages and the token exchange this page relies on.

## What you build

```
┌──────────── Alaya (menu tab) ────────────┐
│  <iframe src="https://your-plugin.host/"> │
│     index.html  (static page)             │  1. ALAYA_INIT / ALAYA_READY
└───────┬───────────────────────────────────┘
        │ 2. POST /api/alaya-token { apiurl, client, username, nonce }
        ▼
  Backend function on the same host   ← holds your DevKey as a secret
        │ 3. POST {apiurl}token  (nonce + DevKey)
        ▼
  Alaya REST API  → access token → back to the page
```

| Part | Where it runs | What it does |
|----|----|----|
| `index.html` | The user's browser, inside Alaya's iframe | Answers `ALAYA_INIT` with `ALAYA_READY`, shows your screen, calls the Alaya API with the token. |
| `/api/alaya-token` | A Vercel Function or a Cloudflare Worker | Exchanges the nonce for an access token. It is the only code that sees your DevKey. |
| Settings | The hosting service | The DevKey (a secret) and the list of Alaya API URLs your backend may call. |

The backend is what keeps the DevKey out of the browser. Never let the
assistant put the DevKey in the page, in a file of the project or in
source control.

## What you need

- Your **DevKey** from the Developer Portal (see
  [Plugin, Step 1](plugin.md#building-a-plugin-step-by-step)).
- An Alaya client with the **Integration API addon (BSMA13)** and an
  **Admin** user, as for any Plugin.
- An account on **Vercel** or **Cloudflare**. Both have free plans; check
  their current limits and terms.
- An AI assistant that can use remote MCP servers, or that can run
  commands in a terminal (for example Claude Code).

## Connect your assistant to the hosting service

The assistant acts with your hosting account, so it can create, change
and delete projects. Turn on confirmation of tool calls in your
assistant, and read what each call does before you approve it.

### Vercel

| Option | How |
|----|----|
| Vercel MCP server | Add the remote server `https://mcp.vercel.com` to your assistant and sign in with your Vercel account. In Claude Code: `claude mcp add --transport http vercel https://mcp.vercel.com`, then run `/mcp` to sign in. Its tools can deploy files, list deployments and read build logs. |
| Vercel CLI | `npm i -g vercel`, then `vercel login`. The assistant deploys with `vercel deploy --prod` and sets settings with `vercel env add`. |

See Vercel's documentation (*Use Vercel's MCP server*) for the current
tool list; the server is in beta and changes.

### Cloudflare

| Option | How |
|----|----|
| Cloudflare MCP servers | Add `https://mcp.cloudflare.com/mcp` (the Cloudflare API) and, for reference, `https://docs.mcp.cloudflare.com/mcp` (the documentation). Both sign in with your Cloudflare account. |
| Wrangler CLI | `npm i -D wrangler`, then `npx wrangler login`. The assistant deploys with `npx wrangler deploy` and sets secrets with `npx wrangler secret put`. |

Cloudflare recommends **Workers with static assets** for new projects
rather than Cloudflare Pages. The example below uses a Worker; Pages
with Pages Functions works the same way if you already use it.

## Give the assistant the Alaya documentation

The assistant can't sign in to the Developer Portal. Save this page and
the [Plugin](plugin.md) page as Markdown files in your project (for
example `docs/alaya-plugin.md` and `docs/alaya-plugin-hosting.md`) and
tell it to read them first. An example request:

```
Build an Alaya Plugin that <describe the screen>.
Read docs/alaya-plugin.md and docs/alaya-plugin-hosting.md first.

- public/index.html: register the message listener in <head>, answer
  ALAYA_INIT with ALAYA_READY immediately, then get a token from
  /api/alaya-token and <what the screen loads and saves>.
- /api/alaya-token: read the DevKey from the ALAYA_DEVKEY secret, only
  call Alaya API URLs listed in ALAYA_API_URLS, return the access token.
- Allow framing only by https://<your Alaya site>.
- Deploy to <Vercel production | a Cloudflare Worker> and give me the
  https URL. Don't ask me for the DevKey; tell me how to set the secret.
```

Set the DevKey yourself, in the hosting service's dashboard or with its
CLI in your own terminal, so that it never passes through the chat.

## The page

The page follows [Plugin, Step 5](plugin.md#building-a-plugin-step-by-step),
and asks your backend for the token instead of calling `{apiurl}token`
itself:

```js
// after ALAYA_INIT has been answered with ALAYA_READY (alaya = msg.data)
async function getAlayaToken() {
  const reply = await fetch('/api/alaya-token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      apiurl: alaya.apiurl,
      client: alaya.client,
      username: alaya.username,
      nonce: alaya.nonce,
    }),
  });
  if (!reply.ok) throw new Error('Could not get an Alaya token');
  return reply.json(); // { token_type, access_token, expires_in }
}

async function callAlaya(path, body) {
  const reply = await fetch(alaya.apiurl + path, {
    method: body ? 'POST' : 'GET',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': token.token_type + ' ' + token.access_token,
    },
    body: body ? JSON.stringify(body) : undefined,
  });
  return reply.json(); // { Result, Message, Obj }
}
```

Send `ALAYA_READY` **before** calling the backend. A function that has
not run for a while can take a few seconds to start, and Alaya only
waits 10 seconds for `ALAYA_READY`.

## The backend function

The function receives the nonce, adds the DevKey and calls the Alaya
token endpoint. It must only send the DevKey to Alaya API URLs you
listed: `apiurl` comes from the browser, and a function that sends the
DevKey to any address it is given hands the DevKey to whoever calls it.

Settings:

| Name | Kind | Value |
|----|----|----|
| `ALAYA_DEVKEY` | Secret | Your DevKey. |
| `ALAYA_API_URLS` | Plain setting | The `apiurl` of each Alaya client that uses the Plugin, comma-separated, exactly as `ALAYA_INIT` sends it (open the menu with **Test Load** and log `alaya.apiurl` to see it). |

### On Vercel

```
public/index.html
api/alaya-token.js
package.json        { "type": "module" }
vercel.json
```

```js
// api/alaya-token.js
const allowed = (process.env.ALAYA_API_URLS || '').split(',').map(u => u.trim()).filter(Boolean);

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  const { apiurl, client, username, nonce } = req.body || {};
  if (!allowed.includes(apiurl)) return res.status(400).json({ error: 'Unknown Alaya API URL' });

  const reply = await fetch(apiurl + 'token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'password', clientid: client, username, nonce, devkey: process.env.ALAYA_DEVKEY,
    }),
  });
  const token = await reply.json();
  if (!reply.ok) return res.status(401).json({ error: token.error_description || 'Token request failed' });
  res.json({ token_type: token.token_type, access_token: token.access_token, expires_in: token.expires_in });
}
```

`vercel.json`:

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "Content-Security-Policy", "value": "frame-ancestors https://your-alaya-site.example.com" }
      ]
    }
  ]
}
```

Settings: `vercel env add ALAYA_DEVKEY production` (mark it sensitive)
and `vercel env add ALAYA_API_URLS production`, then deploy with
`vercel deploy --prod`.

### On Cloudflare

```
public/index.html
public/_headers
src/index.js
wrangler.jsonc
```

```js
// src/index.js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname === '/api/alaya-token' && request.method === 'POST') {
      return alayaToken(await request.json(), env);
    }
    return env.ASSETS.fetch(request); // public/
  },
};

async function alayaToken({ apiurl, client, username, nonce }, env) {
  const allowed = (env.ALAYA_API_URLS || '').split(',').map(u => u.trim()).filter(Boolean);
  if (!allowed.includes(apiurl)) return Response.json({ error: 'Unknown Alaya API URL' }, { status: 400 });

  const reply = await fetch(apiurl + 'token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'password', clientid: client, username, nonce, devkey: env.ALAYA_DEVKEY,
    }),
  });
  const token = await reply.json();
  if (!reply.ok) return Response.json({ error: token.error_description || 'Token request failed' }, { status: 401 });
  return Response.json({ token_type: token.token_type, access_token: token.access_token, expires_in: token.expires_in });
}
```

```jsonc
// wrangler.jsonc
{
  "name": "my-alaya-plugin",
  "main": "src/index.js",
  "compatibility_date": "2026-01-01",
  "assets": { "directory": "./public", "binding": "ASSETS" },
  "vars": { "ALAYA_API_URLS": "https://api.example.com/v3/" }
}
```

```
# public/_headers
/*
  Content-Security-Policy: frame-ancestors https://your-alaya-site.example.com
```

Settings: `npx wrangler secret put ALAYA_DEVKEY`, then deploy with
`npx wrangler deploy`. The page is published at
`https://my-alaya-plugin.<your-subdomain>.workers.dev`, or at a custom
domain you add to the Worker.

## Put the page in a Plugin menu

1. Take the **production** address the assistant gives you. On Vercel,
   preview deployments ask for a Vercel login by default and can't be
   shown inside Alaya; use the project's production domain.
2. Open the address in a browser tab once to check that it loads.
3. In the Module Editor, set it as the menu's **External URL** and test
   it with **Test Load** and **Test View As**, as described in
   [Plugin, Steps 6 and 7](plugin.md#building-a-plugin-step-by-step).
   A program or an assistant that has an Alaya API token can also create
   the module and the Plugin menu with its External URL through the
   [Module Editor API](module_editor_api.md).

When you move the page to another address (a custom domain, a new
project), change the External URL and accept the Terms & Conditions
again.

## Storing data

Keep the Plugin's data in Alaya where you can: add **Master Data** menus
to the same module and read and write them with the `Udf` API (see
[Reading and writing Alaya data](plugin.md#reading-and-writing-alaya-data)).
The data then stays with the client, follows Alaya's company and
user-group rules, and is included when the module is exported.

If you store data with the hosting service instead (a database, a
key-value store, file storage), you hold Alaya users' data on your own
account: collect only what you need and handle it as the Terms &
Conditions and the applicable data-protection law require.

## Before you release

- Read the code the assistant wrote, especially the backend function:
  the DevKey is only read from the secret, and only listed API URLs are
  called.
- Search the project and its history for the DevKey. If it was ever
  committed or pasted somewhere public, ask for a new one.
- Check permissions in the page with `menuInfo` (see
  [Access rights](plugin.md#access-rights)), and remember that the Alaya
  API checks the user's own rights on every call.
- Keep the assistant's access to the hosting account to what it needs,
  and remove it when you are done.
- When the Plugin is used by more clients, add each client's `apiurl`
  to `ALAYA_API_URLS` and its Alaya site to `frame-ancestors`.

## Related Pages

- [Plugin (Blank Menu)](plugin.md)
- [Module Editor](module_editor.md)
- [Module Editor API](module_editor_api.md)
- [API Access](api_access.md)
