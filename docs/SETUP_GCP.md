# Google Cloud setup — OAuth Client for the connector

This is the trickiest step of the install. Allow ~10 minutes the first time. You only do it once.

You will create a Google Cloud project, enable the Search Console API, configure an OAuth consent screen, and create an OAuth Client ID. You end up with two values: a **Client ID** and a **Client Secret**, which you push as Cloudflare Worker secrets.

## 1. Create (or pick) a Google Cloud project

1. Go to https://console.cloud.google.com
2. Top bar → project dropdown → **New Project**
3. Name it `gsc-mcp-connector` (or anything). → **Create**
4. Make sure the new project is selected in the top bar before continuing.

## 2. Enable the Search Console API

1. Left menu → **APIs & Services** → **Library**
2. Search `Search Console API`
3. Click the result → **Enable**

## 3. Configure the OAuth consent screen

> **Why this exists**: when a user grants the connector access to their GSC data, Google shows a consent screen. You configure what's displayed there.

1. Left menu → **APIs & Services** → **OAuth consent screen** (or **Audience** in the new GCP UI)
2. **User Type** :
   - **External** → recommended (works for any Google/Gmail account, including yours)
   - **Internal** → only works if you have a Google Workspace organization
3. → **Create**
4. **App information** :
   - **App name** : `gsc-mcp-connector`
   - **User support email** : your email
5. **Developer contact** : your email
6. → **Save and continue**
7. **Scopes** : skip — we request scopes at runtime, no need to declare them here. → **Save and continue**
8. **Test users** : **+ Add users** → enter the **Gmail address** of every person who will use the connector (including yourself). Each user must be in this list while the app is in *Testing* mode.
   - Up to 100 test users allowed
   - You can add more later
9. → **Save and continue** → **Back to dashboard**
10. **Publishing status** → click **PUBLISH APP** to switch from *Testing* to *In production*. **Do not** leave it on *Testing*.

> **Why you must publish (even for personal use)** — Google enforces **two** restrictions on *Testing*-mode apps :
>
> 1. Only users in your *Test users* list can authenticate.
> 2. **Refresh tokens expire after 7 days.**
>
> The second point is the killer for this connector : every tool call refreshes a Google access token from the stored refresh token. A *Testing*-mode app will silently work for the first week, then start returning `invalid_grant` on every call until you re-do the OAuth flow. Publishing the app makes refresh tokens durable (they last until you revoke them, the user changes their password, or 6 months without use).
>
> **You don't need Google verification.** Publishing an app and getting it verified are two different things. After clicking *PUBLISH APP*, your app sits in *In production / Unverified* — the consent screen shows an "unverified app" warning, you click *Advanced → Go to … (unsafe)*, and it works exactly like a verified app. Verification is only required if you onboard hundreds of users or list the app publicly. For personal or small-team use with the `webmasters.readonly` scope, *unverified production* is the correct setting.

## 4. Create the OAuth Client ID

1. Left menu → **APIs & Services** → **Credentials**
2. **+ Create credentials** → **OAuth client ID**
3. **Application type** : **Web application**
4. **Name** : `gsc-mcp-connector`
5. **Authorized redirect URIs** → **+ Add URI** → paste **exactly** :
   ```
   https://YOUR-WORKER-URL/oauth/google/callback
   ```
   Replace `YOUR-WORKER-URL` with your actual deployed Worker URL (e.g. `gsc-mcp-connector.yourname.workers.dev`).
   - **Critical** : the URL must end with `/oauth/google/callback`. Any typo or truncation = `redirect_uri_mismatch` error during login.
6. → **Create**

A modal appears with two values. **Copy them both** — you'll need them in the next step:

- **Client ID** — long string ending in `.apps.googleusercontent.com`
- **Client Secret** — shorter string starting with `GOCSPX-`

You can also download a JSON file containing both. Either way, keep these safe — they're equivalent to a password.

## 5. Push them as Cloudflare Worker secrets

```bash
npx wrangler secret put GOOGLE_OAUTH_CLIENT_ID
# paste the client_id when prompted

npx wrangler secret put GOOGLE_OAUTH_CLIENT_SECRET
# paste the client_secret when prompted
```

Or via the Cloudflare dashboard (recommended for non-CLI users) :
- **Workers & Pages** → your worker → **Settings** → **Variables and Secrets**
- Add `GOOGLE_OAUTH_CLIENT_ID` (type **Secret**, paste value)
- Add `GOOGLE_OAUTH_CLIENT_SECRET` (type **Secret**, paste value)

## 6. (separately) Set the connector access key

The connector also requires a `MCP_BEARER_TOKEN` secret — a static string you choose. Anyone trying to use the connector must paste this string before being redirected to Google. It's the gate that prevents random people who discover your Worker URL from going through your OAuth Client.

```bash
# Generate one
openssl rand -hex 32

# Push as secret
npx wrangler secret put MCP_BEARER_TOKEN
```

Save this value — you'll paste it in the connector's login UI later (and into ChatGPT/Claude as the connector's "Access key").

## Common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `redirect_uri_mismatch` | The URI configured in step 4 doesn't match what the Worker sends | Re-check the URI ends exactly with `/oauth/google/callback` |
| `403 access_denied` after Google login | The Gmail you logged in with isn't in *Test users* | Step 3 step 8 → add the email |
| `Search Console API has not been enabled` | API not enabled in this project | Step 2 |
| `redirect_uri must use HTTPS` | You used `http://localhost` | OAuth Web clients require HTTPS in redirect URIs (use the deployed Worker URL, not localhost) |
| `invalid_grant: account not found` | The OAuth client was deleted in GCP | Re-create in step 4, push new secrets |
| `invalid_grant` after ~7 days of working fine | OAuth app left in *Testing* mode → Google silently expired the refresh token | Step 3.10 → switch *Publishing status* to *In production*, then disconnect and reconnect the connector to issue a fresh (durable) refresh token |
| `No Google refresh token in grant. Re-authorize` | Old grant from a previous flow version | Delete the connector in ChatGPT/Claude and recreate it |
| `Invalid access key. Try again.` despite the value matching what you set | On Windows, `wrangler secret put` was fed via a PowerShell pipe that appended `\r\n` or a UTF-8 BOM to the stored secret | Re-set the secret using the interactive prompt or the Cloudflare dashboard (see README section 4, "Windows secret pitfall") |
