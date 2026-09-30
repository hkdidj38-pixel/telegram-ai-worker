# telegram-ai-worker
Telegram bot with AI replies, running on Cloudflare Workers. Serverless, webhook-based, deploys in minutes.
# telegram-ai-worker

Telegram bot with AI replies, running on Cloudflare Workers. No server, no hosting bill for small usage.

## Stack
- Cloudflare Workers
- Workers AI (binding, no external API key needed)
- Telegram Bot API (webhook)

## Setup

1. Create a bot with @BotFather, copy the token.
2. Clone and install:
   ```
   git clone https://github.com/YOUR_USER/telegram-ai-worker
   cd telegram-ai-worker
   npm install
   ```
3. Set secrets:
   ```
   npx wrangler secret put BOT_TOKEN
   npx wrangler secret put WEBHOOK_SECRET
   ```
4. Deploy:
   ```
   npx wrangler deploy
   ```
5. Point Telegram at your worker:
   ```
   curl "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook" \
     -d "url=https://<your-worker>.workers.dev" \
     -d "secret_token=<WEBHOOK_SECRET>"
   ```

## wrangler.toml
```toml
name = "telegram-ai-worker"
main = "src/index.js"
compatibility_date = "2025-01-01"

[ai]
binding = "AI"
```

## src/index.js
```js
export default {
  async fetch(request, env) {
    if (request.method !== "POST") return new Response("ok");

    // verify request really comes from Telegram
    const secret = request.headers.get("X-Telegram-Bot-Api-Secret-Token");
    if (secret !== env.WEBHOOK_SECRET) return new Response("no", { status: 403 });

    const update = await request.json();
    const msg = update.message;
    if (!msg?.text) return new Response("ok");

    const ai = await env.AI.run("@cf/meta/llama-3.1-8b-instruct", {
      messages: [
        { role: "system", content: "You are a helpful assistant. Keep replies short." },
        { role: "user", content: msg.text },
      ],
    });

    await fetch(`https://api.telegram.org/bot${env.BOT_TOKEN}/sendMessage`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ chat_id: msg.chat.id, text: ai.response }),
    });

    return new Response("ok");
  },
};
```

## Notes
- Model names change. Check the Workers AI model list before deploying.
- Free tier has daily limits on AI usage.
- No memory between messages yet. Add KV or D1 if you want history.
- Never commit your bot token.

## License
MIT
