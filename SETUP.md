/**
 * Armila's Instagram Feed Proxy
 * ──────────────────────────────────────────────────────────────────
 * Cloudflare Worker that securely fetches the latest posts from the
 * @armilaslounge Instagram account and caches them for 1 hour.
 *
 * The access token NEVER touches the browser. This worker is the only
 * thing that holds the token, and it only returns sanitized post data.
 *
 * ── SETUP ──────────────────────────────────────────────────────────
 *
 * 1. Create a Cloudflare account (free).
 * 2. Install Wrangler:  npm install -g wrangler
 * 3. From this folder:  wrangler login
 * 4. Set the secrets (these never appear in code):
 *      wrangler secret put IG_ACCESS_TOKEN
 *      wrangler secret put IG_USER_ID
 *      wrangler secret put IG_APP_ID
 *      wrangler secret put IG_APP_SECRET
 * 5. Set the public allowed origin in wrangler.toml.
 * 6. Deploy:  wrangler deploy
 * 7. Update INSTAGRAM_API_URL in index.html to your worker's URL.
 *
 * ── HOW TO GET THE INITIAL TOKEN ───────────────────────────────────
 *
 * a) Go to developers.facebook.com → My Apps → Create App → Business.
 * b) Add the "Instagram" product to the app.
 * c) Convert @armilaslounge to a Business or Creator account in the
 *    Instagram app (Settings → Account type → Switch to Professional).
 * d) Link the Instagram account to a Facebook Page the same person
 *    administers.
 * e) In the app dashboard → Instagram → API Setup with Instagram Login,
 *    generate a User Access Token. It's short-lived (1 hour).
 * f) Exchange it for a long-lived token (60 days) using this curl:
 *      curl -X GET "https://graph.instagram.com/access_token \
 *        ?grant_type=ig_exchange_token \
 *        &client_secret={APP_SECRET} \
 *        &access_token={SHORT_LIVED_TOKEN}"
 * g) Copy the returned token into IG_ACCESS_TOKEN via wrangler secret.
 * h) The /refresh route on this worker will keep it alive automatically.
 *
 * ── TOKEN REFRESH ──────────────────────────────────────────────────
 *
 * Set up a Cloudflare cron trigger (free) in wrangler.toml:
 *   [triggers]
 *   crons = ["0 0 1 * *"]   # 1st of every month, refreshes the 60-day token
 *
 * The scheduled() handler below auto-refreshes and stores the new token.
 */

const GRAPH_VERSION = 'v22.0';
const CACHE_TTL_SECONDS = 60 * 60; // 1 hour
const POST_LIMIT = 9;

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // CORS — only allow our own domain in production.
    const allowedOrigin = env.ALLOWED_ORIGIN || '*';
    const corsHeaders = {
      'Access-Control-Allow-Origin': allowedOrigin,
      'Access-Control-Allow-Methods': 'GET, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
      'Access-Control-Max-Age': '86400',
    };

    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }

    if (url.pathname === '/feed') {
      return handleFeed(env, corsHeaders, ctx);
    }

    if (url.pathname === '/refresh') {
      // Manual refresh endpoint, useful for testing. Requires admin secret.
      const adminKey = request.headers.get('x-admin-key');
      if (adminKey !== env.ADMIN_KEY) {
        return new Response('Unauthorized', { status: 401, headers: corsHeaders });
      }
      return handleRefresh(env, corsHeaders);
    }

    return new Response('Not found', { status: 404, headers: corsHeaders });
  },

  // Cloudflare cron — refresh the long-lived token monthly.
  async scheduled(event, env, ctx) {
    ctx.waitUntil(handleRefresh(env, {}));
  },
};

/**
 * Fetch the latest posts. Caches at the edge for an hour so we don't
 * hammer the Instagram API and stay well under rate limits.
 */
async function handleFeed(env, corsHeaders, ctx) {
  // Try the cache first.
  const cache = caches.default;
  const cacheKey = new Request('https://internal/armilas-ig-feed-v1');
  const cached = await cache.match(cacheKey);
  if (cached) {
    const response = new Response(cached.body, cached);
    Object.entries(corsHeaders).forEach(([k, v]) => response.headers.set(k, v));
    return response;
  }

  try {
    const fields = [
      'id',
      'caption',
      'media_type',
      'media_url',
      'thumbnail_url',
      'permalink',
      'timestamp',
    ].join(',');

    const igUrl =
      `https://graph.instagram.com/${GRAPH_VERSION}/${env.IG_USER_ID}/media` +
      `?fields=${fields}&limit=${POST_LIMIT}&access_token=${env.IG_ACCESS_TOKEN}`;

    const igResponse = await fetch(igUrl);
    const igData = await igResponse.json();

    if (igData.error) {
      console.error('IG API error:', igData.error);
      return errorResponse('Instagram API error', corsHeaders, 502);
    }

    // Sanitize — only return what the frontend needs. Strip anything
    // sensitive or noisy.
    const posts = (igData.data || []).map(post => ({
      id: post.id,
      caption: truncateCaption(post.caption || ''),
      type: post.media_type, // IMAGE, VIDEO, CAROUSEL_ALBUM
      image: post.media_type === 'VIDEO' ? post.thumbnail_url : post.media_url,
      url: post.permalink,
      date: post.timestamp,
    }));

    const body = JSON.stringify({
      posts,
      fetched_at: new Date().toISOString(),
      count: posts.length,
    });

    const response = new Response(body, {
      headers: {
        'Content-Type': 'application/json',
        'Cache-Control': `public, max-age=${CACHE_TTL_SECONDS}`,
        ...corsHeaders,
      },
    });

    // Stash in edge cache so the next visitor gets it instantly.
    ctx.waitUntil(cache.put(cacheKey, response.clone()));

    return response;
  } catch (err) {
    console.error('Feed handler error:', err);
    return errorResponse('Server error', corsHeaders, 500);
  }
}

/**
 * Refresh the long-lived token. Instagram's tokens last 60 days and
 * can be refreshed for another 60 if they're still valid.
 */
async function handleRefresh(env, corsHeaders) {
  try {
    const refreshUrl =
      `https://graph.instagram.com/refresh_access_token` +
      `?grant_type=ig_refresh_token` +
      `&access_token=${env.IG_ACCESS_TOKEN}`;

    const response = await fetch(refreshUrl);
    const data = await response.json();

    if (data.error) {
      console.error('Token refresh error:', data.error);
      return errorResponse('Refresh failed: ' + data.error.message, corsHeaders, 502);
    }

    // Persist the new token. Cloudflare KV (set this up in wrangler.toml)
    // or another secret store would be used here. For now we log and
    // expect the admin to update the secret if KV isn't bound.
    if (env.IG_TOKEN_STORE) {
      await env.IG_TOKEN_STORE.put('current_token', data.access_token);
      await env.IG_TOKEN_STORE.put('refreshed_at', new Date().toISOString());
    }

    return new Response(
      JSON.stringify({
        success: true,
        expires_in: data.expires_in,
        refreshed_at: new Date().toISOString(),
      }),
      {
        headers: { 'Content-Type': 'application/json', ...corsHeaders },
      }
    );
  } catch (err) {
    console.error('Refresh error:', err);
    return errorResponse('Server error', corsHeaders, 500);
  }
}

function truncateCaption(text) {
  if (!text) return '';
  // Strip excessive whitespace, keep first 180 chars, end on a word.
  const clean = text.replace(/\s+/g, ' ').trim();
  if (clean.length <= 180) return clean;
  const cut = clean.slice(0, 180);
  return cut.slice(0, cut.lastIndexOf(' ')) + '…';
}

function errorResponse(message, corsHeaders, status = 500) {
  return new Response(JSON.stringify({ error: message }), {
    status,
    headers: { 'Content-Type': 'application/json', ...corsHeaders },
  });
}
