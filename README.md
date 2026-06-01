/**
 * Cloudflare Worker: 胡蝶蘭配送アプリ用
 *
 * 機能:
 *   1. /send (POST): Resendへメール転送 (API キー秘匿)
 *   2. /sync (POST): ドライバー状態をKVに保存
 *   3. /sync (GET):  管理者がドライバー状態をKVから取得
 *   4. /sync/list (GET): 保存中の全routeId一覧 (管理画面用)
 *
 * 必要な環境変数 (Cloudflare ダッシュボードで設定):
 *   - RESEND_API_KEY : Resend で発行した API キー (re_xxxxx...) [必須:メール送信時]
 *   - AUTH_TOKEN     : クライアントから X-Auth-Token ヘッダーで送る共有秘密 [推奨]
 *
 * 必要な KV Namespace バインディング:
 *   - ROUTES_KV : ドライバー作業状態保存用 [必須:同期機能時]
 *
 * デプロイ手順:
 *   1. Cloudflareダッシュボード → Workers & Pages → Create
 *   2. "Hello World" テンプレートで作成
 *   3. このファイル内容をエディタに貼り付けて保存
 *   4. Settings → Variables → Environment Variables で
 *      RESEND_API_KEY と AUTH_TOKEN を Secret 設定で追加
 *   5. Settings → KV Namespace Bindings で ROUTES_KV を追加
 *   6. デプロイ → 生成されたURLを本アプリの「⑧ Worker URL」に貼り付け
 *
 * エンドポイント (本アプリは自動で適切なパスに送信):
 *   POST {WorkerURL}/send       → メール送信
 *   POST {WorkerURL}/sync       → ドライバー状態をKVに保存 (?routeId=...)
 *   GET  {WorkerURL}/sync       → 状態取得 (?routeId=...)
 *   GET  {WorkerURL}/sync/list  → 保存中の全routeId一覧
 *
 * 後方互換: 旧版アプリ(パス指定なし)はPOSTを/sendとして処理
 */

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const path = url.pathname;

    // CORS preflight
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        status: 204,
        headers: corsHeaders(),
      });
    }

    // 認証トークンチェック (環境変数 AUTH_TOKEN が設定されている場合のみ)
    if (env.AUTH_TOKEN) {
      const provided = request.headers.get('X-Auth-Token');
      if (provided !== env.AUTH_TOKEN) {
        return jsonResp({ error: 'Unauthorized' }, 401);
      }
    }

    try {
      // ========== /sync ルート ==========
      if (path === '/sync' || path === '/sync/') {
        if (!env.ROUTES_KV) {
          return jsonResp({ error: 'ROUTES_KV namespace is not bound in Worker' }, 500);
        }
        const routeId = url.searchParams.get('routeId');
        if (!routeId) {
          return jsonResp({ error: 'routeId query parameter is required' }, 400);
        }

        if (request.method === 'POST') {
          // ドライバー状態を保存 (30日後自動削除)
          const data = await request.text();
          await env.ROUTES_KV.put(routeId, data, {
            expirationTtl: 60 * 60 * 24 * 30, // 30日
            metadata: { updated: new Date().toISOString() }
          });
          return jsonResp({ ok: true, routeId, saved: new Date().toISOString() });
        }

        if (request.method === 'GET') {
          // 管理者: 状態取得
          const data = await env.ROUTES_KV.get(routeId);
          if (!data) {
            return jsonResp({ routeId, state: null, exists: false });
          }
          return new Response(data, {
            status: 200,
            headers: { 'Content-Type': 'application/json', ...corsHeaders() }
          });
        }

        return jsonResp({ error: 'Method not allowed for /sync' }, 405);
      }

      // ========== /sync/list ルート ==========
      if (path === '/sync/list' || path === '/sync/list/') {
        if (!env.ROUTES_KV) {
          return jsonResp({ error: 'ROUTES_KV namespace is not bound' }, 500);
        }
        if (request.method !== 'GET') {
          return jsonResp({ error: 'GET only' }, 405);
        }
        const list = await env.ROUTES_KV.list({ limit: 100 });
        const items = list.keys.map(k => ({
          routeId: k.name,
          updated: k.metadata?.updated || null,
          expiration: k.expiration || null
        }));
        return jsonResp({ items, count: items.length });
      }

      // ========== /send ルート (Resend中継) + 後方互換 (旧版は / を /send 扱い) ==========
      if (path === '/send' || path === '/send/' || path === '/' || path === '') {
        if (request.method !== 'POST') {
          return new Response('Method Not Allowed', {
            status: 405,
            headers: corsHeaders()
          });
        }
        return await handleSendMail(request, env);
      }

      return jsonResp({ error: `Unknown path: ${path}` }, 404);

    } catch (e) {
      console.error('Worker error:', e);
      return jsonResp({ error: `Server error: ${e.message}` }, 500);
    }
  },
};

// ========== Resend メール送信処理 ==========
async function handleSendMail(request, env) {
  if (!env.RESEND_API_KEY) {
    return jsonResp({ error: 'RESEND_API_KEY is not set in Worker environment' }, 500);
  }

  let payload;
  try {
    payload = await request.json();
  } catch (e) {
    return jsonResp({ error: 'Invalid JSON body' }, 400);
  }

  if (!payload.from || !payload.to || !payload.subject) {
    return jsonResp({ error: 'Missing required fields: from, to, subject' }, 400);
  }

  try {
    const resendResp = await fetch('https://api.resend.com/emails', {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${env.RESEND_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(payload),
    });
    const respBody = await resendResp.text();
    return new Response(respBody, {
      status: resendResp.status,
      headers: { 'Content-Type': 'application/json', ...corsHeaders() },
    });
  } catch (e) {
    return jsonResp({ error: `Resend API call failed: ${e.message}` }, 502);
  }
}

// ========== ユーティリティ ==========
function jsonResp(obj, status = 200) {
  return new Response(JSON.stringify(obj), {
    status,
    headers: { 'Content-Type': 'application/json', ...corsHeaders() }
  });
}

function corsHeaders() {
  return {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'POST, GET, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, X-Auth-Token',
    'Access-Control-Max-Age': '86400',
  };
}
