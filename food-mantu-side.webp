# Instagram Feed Setup Guide

This is the one-time setup to make the Instagram section on the website show real, live posts from @armilaslounge.

Total setup time: about 45 minutes, mostly spent in Meta's developer console.

---

## What you're building

```
Website  ──→  Cloudflare Worker  ──→  Instagram Graph API
(public)      (holds the token)       (your account)
```

The website never sees the access token. The worker is a tiny piece of code that lives on Cloudflare's free tier, holds the secret token, talks to Instagram, and hands back clean post data to the website. The worker also auto-refreshes the token every month so it never expires.

---

## Step 1 — Convert the Instagram account to Professional

In the Instagram mobile app, on the @armilaslounge account:

1. Settings → Account → Account type and tools → Switch to professional account.
2. Pick **Business** (not Creator). "Restaurant" is the right category.
3. When prompted, you can skip connecting to Facebook for now, or connect it (we'll need it linked in Step 2 either way).

This is required. Personal Instagram accounts cannot be read by the API.

---

## Step 2 — Link to a Facebook Page

The Graph API requires the Instagram account to be linked to a Facebook Page that the same person administers.

1. If Armila's doesn't have a Facebook Page yet, create one at facebook.com/pages/create. Pick **Restaurant** as the category and fill in the basics.
2. On the Facebook Page → Settings → Linked accounts → Instagram → Connect account. Log in with the @armilaslounge credentials.

This is a one-time link. After this, the Facebook Page and Instagram account are tied together for API purposes.

---

## Step 3 — Create the Meta App

1. Go to developers.facebook.com → My Apps → Create app.
2. Use case: **Other**.
3. App type: **Business**.
4. App name: "Armila's Website" (only you see this).
5. Contact email: info@armilaslounge.com.
6. Business portfolio: pick the one tied to the Facebook Page from Step 2 (create one if needed).

Once the app exists, in the left sidebar: **Add product** → **Instagram** → Set up.

Choose **API setup with Instagram Login** (the recommended path).

---

## Step 4 — Generate the Access Token

In the Instagram product dashboard for your new app:

1. Under "1. Generate access tokens", click **Add account**.
2. Log in with the Armila's Facebook account.
3. Select the @armilaslounge Instagram account.
4. Grant all requested permissions (read posts, profile info).
5. The dashboard will show you a short-lived token (valid 1 hour). Copy it.

Now exchange it for a long-lived token (valid 60 days). Open a terminal and run, replacing the two placeholders:

```bash
curl -X GET "https://graph.instagram.com/access_token?grant_type=ig_exchange_token&client_secret=YOUR_APP_SECRET&access_token=YOUR_SHORT_LIVED_TOKEN"
```

You'll find your App Secret in the Meta app dashboard → App settings → Basic.

The response looks like:

```json
{
  "access_token": "IGQVJ...long_string...",
  "token_type": "bearer",
  "expires_in": 5183944
}
```

That `access_token` value is what we use. Save it somewhere safe for the next step.

---

## Step 5 — Find your Instagram User ID

Same terminal, run:

```bash
curl -X GET "https://graph.instagram.com/v22.0/me?fields=id,username&access_token=YOUR_LONG_LIVED_TOKEN"
```

Response:

```json
{ "id": "17841405309211844", "username": "armilaslounge" }
```

Save that `id` — that's your `IG_USER_ID`.

---

## Step 6 — Deploy the Cloudflare Worker

The worker code is in `/api/worker.js`. Cloudflare's free tier covers far more traffic than this site will ever see.

1. Sign up at cloudflare.com (free).
2. Install Wrangler (Cloudflare's CLI):
   ```bash
   npm install -g wrangler
   ```
3. From the `/api` folder:
   ```bash
   wrangler login
   ```
   This opens a browser to authorize.
4. Set the secrets (one command at a time, paste the value when prompted):
   ```bash
   wrangler secret put IG_ACCESS_TOKEN
   wrangler secret put IG_USER_ID
   wrangler secret put IG_APP_SECRET
   wrangler secret put ADMIN_KEY
   ```
   For `ADMIN_KEY`, pick any random long string. You'll use it if you ever need to manually trigger a token refresh.
5. (Optional but recommended) Create a KV namespace so the worker can persist refreshed tokens:
   ```bash
   wrangler kv:namespace create "IG_TOKEN_STORE"
   ```
   Copy the returned `id` and paste it into `wrangler.toml` under the `[[kv_namespaces]]` section (uncomment those lines).
6. Open `wrangler.toml` and update `ALLOWED_ORIGIN` to your real domain (probably `https://armilaslounge.com`).
7. Deploy:
   ```bash
   wrangler deploy
   ```

You'll get a URL back, something like `https://armilas-instagram.your-subdomain.workers.dev`.

Test it:

```bash
curl https://armilas-instagram.your-subdomain.workers.dev/feed
```

You should see real Instagram posts as JSON.

---

## Step 7 — Wire it into the website

Open `index.html`. Find this line near the bottom of the page:

```javascript
const INSTAGRAM_API_URL = 'https://YOUR-WORKER-SUBDOMAIN.workers.dev/feed';
```

Replace the placeholder with the URL you got from Step 6. Save, push to GitHub Pages, and the section will fill in with live posts the next time someone visits.

---

## Ongoing maintenance

**Nothing.** Seriously. The worker has a cron trigger that refreshes the token on the 1st of every month. As long as Armila's keeps posting to Instagram, the website will keep pulling the latest 9 posts. Posts are cached at Cloudflare's edge for 1 hour, so you'll hit the Instagram API at most ~24 times a day — far below the 200/hour limit.

The only thing that would break this:
- Someone changing the @armilaslounge password without updating the token. If that happens, redo Step 4 to get a fresh token.
- Switching the Instagram account back to Personal. Don't do that.
- Disconnecting the Instagram account from the Facebook Page. Don't do that either.

---

## If something breaks

The website is built to fail gracefully. If the Instagram API is down, the token expires, or anything else goes wrong, the website will show a quiet "Follow us on Instagram" card instead of broken thumbnails. Nothing visible breaks for guests.

To debug, hit the worker directly in a browser:
```
https://armilas-instagram.your-subdomain.workers.dev/feed
```

If you see an error JSON, the message will tell you what's wrong (expired token, account permissions changed, etc.).

To manually trigger a token refresh:
```bash
curl -H "x-admin-key: YOUR_ADMIN_KEY" https://armilas-instagram.your-subdomain.workers.dev/refresh
```
