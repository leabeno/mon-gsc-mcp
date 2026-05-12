# gsc-mcp-connector

> Self-hosted Google Search Console MCP server, deployable to Cloudflare Workers in 15 minutes. Plug it into ChatGPT, Claude, or any MCP-capable client and query your GSC data in natural language.

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/JuJu78/gsc-mcp-connector)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## What this gives you

Once deployed, you get a private MCP endpoint that exposes 4 tools to your AI assistant:

- `list_sites` — discover every property accessible to the authenticated user
- `query_search_analytics` — clicks, impressions, CTR, position, filterable by query / page / country / device / search appearance / date
- `inspect_url` — full URL Inspection API output (indexing status, canonical, mobile, AMP)
- `list_sitemaps` — every submitted sitemap and its processing status

You ask: *"Quelles sont mes 50 requêtes avec la plus grosse perte de clics entre les 28 derniers jours et les 28 jours précédents ?"* and the assistant pulls the data, computes the delta, and writes the analysis. No more SQL exports.

## How it works

```
ChatGPT/Claude  ──OAuth──▶  Your Worker  ──OAuth──▶  Google
                                 │
                                 └─ holds your Google refresh_token
                                    (encrypted, in OAuth grant props)
```

Two OAuth chains :
1. **MCP client → Worker** : ChatGPT/Claude do OAuth 2.1 + PKCE against your Worker. The login UI asks for a static "access key" you set at deploy time (the connector gate).
2. **Worker → Google** : after the access key check, the user is redirected to Google's consent screen to grant `webmasters.readonly`. The resulting refresh token is stored in the OAuth grant; on every tool call, the Worker refreshes a fresh access token and calls GSC.

The user's Google account drives access — no Service Account, no GSC user-management dance, no permission propagation delays.

## Prerequisites

| Item | Cost | Required ? |
|---|---|---|
| ChatGPT Plus / Pro / Team **or** Claude.ai Pro / Team | $20/mo+ | Custom MCP connectors are gated on paid plans. |
| Cloudflare account | Free tier is enough | Yes |
| Google Cloud project | Free | Yes — to create an OAuth Client |
| Verified Search Console property | Free | Yes (you already have it) |
| Node.js 20+ + `wrangler` CLI | Free | Recommended for the secret-setting steps |

The Cloudflare Workers free tier (100k requests/day) is largely enough for personal SEO usage. **No paid Cloudflare plan needed.**

## Quick start (~15 min)

### 1. Deploy the Worker

Click the **Deploy to Cloudflare** button at the top of this README. Cloudflare clones the repo into your account, installs deps, and gives you a public URL like `https://gsc-mcp-connector.<your-subdomain>.workers.dev`.

> **Note** : at this stage, you'll be asked for `MCP_BEARER_TOKEN`, `GOOGLE_OAUTH_CLIENT_ID`, and `GOOGLE_OAUTH_CLIENT_SECRET`. You don't have the Google ones yet — fill `MCP_BEARER_TOKEN` with any random hex string for now (you can change later), and paste anything in the two Google fields. We'll set them properly in step 4.
>
> If you'd rather skip the button, clone the repo locally, run `npm install`, then `npx wrangler deploy`.

After deployment, **note your Worker URL**. You will need it both in step 3 and in step 5.

### 2. Generate a connector access key

This is a static random string used as a gate before Google OAuth. Anyone using the connector must paste this string in the login UI.

```bash
openssl rand -hex 32
```

Save the output — you'll set it as a secret and use it in ChatGPT/Claude.

### 3. Set up Google Cloud (OAuth Client)

Follow the step-by-step in [docs/SETUP_GCP.md](docs/SETUP_GCP.md). Use **your Worker URL from step 1** in the Authorized redirect URI. You'll end up with a **Client ID** and a **Client Secret**.

This is the longest step (~10 min the first time) but you only do it once.

### 4. Configure the Worker secrets

```bash
npx wrangler secret put MCP_BEARER_TOKEN
# paste the value from step 2

npx wrangler secret put GOOGLE_OAUTH_CLIENT_ID
# paste the Client ID from step 3

npx wrangler secret put GOOGLE_OAUTH_CLIENT_SECRET
# paste the Client Secret from step 3
```

Or via the Cloudflare dashboard : **Workers & Pages** → your worker → **Settings** → **Variables and Secrets** → add each as type **Secret**.

> **Windows secret pitfall — do not pipe values into `wrangler secret put`.**
> On Windows, both `Get-Content … | wrangler secret put …` and `$value | wrangler secret put …` can silently append a `\r\n` or a UTF-8 BOM to the stored secret. The connector then rejects every login with `401 Invalid access key. Try again.` even though the value looks identical to what you set — the comparison fails on 1–2 invisible bytes. Three safe alternatives :
>
> 1. **Interactive prompt** — run `npx wrangler secret put MCP_BEARER_TOKEN` and *paste the value at the prompt* (no pipe).
> 2. **Cloudflare dashboard** — most reliable for non-CLI users; paste directly in the browser, zero shell encoding involved.
> 3. **File + `cmd /c type`** — if you must script it, write the value to a UTF-8-without-BOM file, then pipe via `cmd` (not PowerShell):
>    ```powershell
>    [System.IO.File]::WriteAllText("$env:TEMP\sec.txt", "your-value", [System.Text.UTF8Encoding]::new($false))
>    cmd /c "type %TEMP%\sec.txt | npx wrangler secret put MCP_BEARER_TOKEN"
>    Remove-Item "$env:TEMP\sec.txt"
>    ```

### 5. Plug into ChatGPT (Plus/Pro/Team)

ChatGPT → **Settings** → **Connectors** → **Add custom connector** :

- **Name** : `gsc`
- **MCP Server URL** : `https://YOUR-WORKER-URL/mcp` (must end with `/mcp`)
- **Authentication** : `OAuth`
- Check "I understand and want to continue"
- **Create**

A popup opens to your Worker's login UI. Paste your `MCP_BEARER_TOKEN` → click **Continue with Google →** → Google asks you to log in (use the account that owns your GSC property) and grant `webmasters.readonly` → you're redirected back to ChatGPT, connector is active.

In a new chat, **enable the `gsc` connector** in the toolbar, then ask: *"List my Google Search Console sites"*. You should see your properties.

### 6. (Optional) Plug into Claude.ai (Pro/Team)

**Settings** → **Integrations** → **Add custom integration** → same URL, same flow.

## Local development

```bash
git clone https://github.com/JuJu78/gsc-mcp-connector
cd gsc-mcp-connector
npm install
cp .dev.vars.example .dev.vars
# edit .dev.vars with your real Client ID + Client Secret + bearer token
npx wrangler dev
```

The dev server runs on `http://localhost:8787`. Note that local dev cannot fully complete the Google OAuth flow because Google's redirect URIs require HTTPS. For a true end-to-end test, deploy to Cloudflare and test against the workers.dev URL.

## Limitations

- **Read-only.** Adding write operations (submit sitemap, request indexing) is left out by design — they're risky in an LLM context. Open a PR if you need them.
- **Single-tenant by design.** One operator deploys, one bearer token gates the connector, the access is bound to whoever completes the Google OAuth dance. Multi-user SaaS-style is out of scope.
- **OAuth consent must be set to *In production*** (no verification needed). Leaving it in *Testing* caps you at 100 test users **and** silently expires the Google refresh token after 7 days, breaking the connector. Publishing the app removes both restrictions; you only need full Google verification if you plan to onboard hundreds of users or list the app publicly. See [docs/SETUP_GCP.md step 3.10](docs/SETUP_GCP.md#3-configure-the-oauth-consent-screen).
- **GSC API quotas** — 1200 queries/min/project, 30k/day. Plenty for interactive use.
- **Date range** — GSC returns the last 16 months. Earlier dates error.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Error 400: redirect_uri_mismatch` during Google login | Authorized redirect URI in GCP doesn't exactly match what the Worker sends | Verify `https://YOUR-WORKER/oauth/google/callback` is configured *as-is* in [GCP Credentials](https://console.cloud.google.com/apis/credentials) |
| `Error 403: access_denied` after Google login | Logged-in account isn't in *Test users* of the OAuth consent screen | Add the Gmail in [OAuth consent screen → Audience → Test users](https://console.cloud.google.com/apis/credentials/consent) |
| `something went wrong` after authorization in ChatGPT | Stale OAuth grant from a previous attempt | Delete the connector in ChatGPT and recreate it |
| `No Google refresh token in grant. Re-authorize` | The grant was created before you deployed v0.4+ | Delete the connector in ChatGPT/Claude and recreate it |
| `invalid_grant` after the connector worked fine for ~7 days | OAuth consent screen left in *Testing* mode → Google silently expired the refresh token | Publish the OAuth app (Publishing status → *In production*), then disconnect and reconnect the connector to issue a fresh refresh token. See [docs/SETUP_GCP.md step 3.10](docs/SETUP_GCP.md#3-configure-the-oauth-consent-screen) |
| `Invalid access key. Try again.` after pasting the bearer | Either the value is wrong, **or** on Windows the secret was set via a PowerShell pipe that appended `\r\n` / a BOM (the comparison fails on 1–2 invisible bytes) | Re-set via interactive `wrangler secret put` or via the Cloudflare dashboard — see step 4 "Windows secret pitfall" above |
| `tools/call list_sites` returns `{}` | Authenticated Google account has no GSC properties (or wrong account) | Verify which account you used in the Google consent step — it must own GSC properties |
| ChatGPT says "no tools available" | URL doesn't end with `/mcp` | Server URL must be `https://YOUR-WORKER/mcp`, not the bare root |

## Stack

- **Cloudflare Workers** + Durable Objects (via [`agents`](https://github.com/cloudflare/agents) SDK ≥0.12)
- [`@cloudflare/workers-oauth-provider`](https://github.com/cloudflare/workers-oauth-provider) for OAuth 2.1 + DCR + PKCE
- [`@modelcontextprotocol/sdk`](https://github.com/modelcontextprotocol/typescript-sdk) for tool definitions
- KV namespace `OAUTH_KV` for OAuth state
- Google OAuth 2.0 flow signed natively via Web Crypto API (no Node deps)

## Credits

Built by [Julien Gourdon](https://julien-gourdon.fr) — SEO consultant exploring the intersection of search and AI.

## License

MIT — see [LICENSE](LICENSE).
