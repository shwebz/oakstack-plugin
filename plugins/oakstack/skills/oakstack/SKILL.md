---
name: oakstack
description: Add scheduled jobs (cron) or reliable webhook receiving to an app using Oakstack. Use when the user wants something to run on a schedule (nightly cleanup, daily digest, sync every N minutes, reminders), wants Vercel Cron or a cron server replaced with something that retries and logs, or receives webhooks from Stripe, GitHub, Shopify, Clerk, or similar and must not lose events during deploys or outages. Also use when the user mentions Oakstack, OAKSTACK_API_KEY, or the oakstack npm package.
---

# Oakstack

Oakstack is one API key for app infrastructure. Two modules are live:

- **clock**: Oakstack calls a URL in the app on a cron schedule, retries failures, and logs every run.
- **hook**: a permanent inbound URL for webhooks. Oakstack stores every event, forwards it to the app with the original headers and body, retries for about 11 hours, and can replay any event.

PDFs (print) and email (post) are coming; don't promise them yet.

Full docs as markdown: https://oakstack.dev/llms.txt. Fetch https://oakstack.dev/docs/clock.md or https://oakstack.dev/docs/hook.md when you need details.

## Before you start

1. **API key.** Check for `OAKSTACK_API_KEY` in the environment or `.env*` files. If it's missing, ask the user to create one at https://oakstack.dev/dashboard/api-keys and add it to `.env.local` (and to their host's environment variables for production). Never ask them to paste the key into chat, and never commit it.
2. **A public URL.** Oakstack only calls public `https://` URLs, never localhost. If the app isn't deployed yet, either deploy first and use the production URL, or use a tunnel (`cloudflared tunnel --url http://localhost:3000`) for testing. Say this up front so the user isn't surprised.
3. **Install the SDK:** `npm install oakstack` (or pnpm, yarn, or bun). It has zero dependencies and works in Node 18+, Bun, Deno, and edge runtimes.

If the Oakstack MCP tools (`clock_create_job`, `hook_create_endpoint`, ...) are available, use them to create jobs and endpoints directly. Otherwise, write a one-off setup script with the SDK (below) and run it.

## Scheduled job (clock)

**Step 1: the route Oakstack will call.** Next.js App Router example. Adapt it to the app's framework.

```ts
// app/api/cron/cleanup/route.ts
import { verifySignature } from "oakstack";

export async function POST(req: Request) {
  const body = await req.text();
  const ok = await verifySignature(
    process.env.OAKSTACK_SIGNING_SECRET!,
    req.headers.get("oakstack-signature"),
    body,
  );
  if (!ok) return new Response("bad signature", { status: 401 });

  // Do the work here. Answer 2xx within 15 seconds; for longer work, answer 202 and
  // continue in the background. Make it safe to run twice (runs can be retried):
  // req.headers.get("oakstack-run-id") is stable across retries of one run.

  return Response.json({ ok: true });
}
```

**Step 2: create the job** (once, from a script or with the MCP tool):

```ts
import { Oakstack } from "oakstack";
const oakstack = new Oakstack(); // reads OAKSTACK_API_KEY

const job = await oakstack.clock.jobs.create(
  {
    name: "Nightly cleanup",
    schedule: "0 3 * * *", // minute hour day-of-month month day-of-week
    timezone: "America/Chicago", // ask the user's time zone if it matters; default UTC
    url: "https://yourapp.com/api/cron/cleanup",
  },
  { idempotencyKey: "nightly-cleanup-v1" }, // makes re-running the script safe
);
console.log(job.id, job.signingSecret);
```

**Step 3:** tell the user to add `OAKSTACK_SIGNING_SECRET=<job.signingSecret>` to the app's environment (local and production). Then test it: `await oakstack.clock.jobs.run(job.id)`, and check `await oakstack.clock.jobs.runs(job.id)` for `status: "succeeded"`.

Cron cheat sheet: `*/5 * * * *` every 5 minutes, `0 * * * *` hourly, `0 9 * * *` daily at 9:00, `0 9 * * 1-5` weekdays at 9:00, `0 0 1 * *` monthly. Five fields only; there are no seconds.

If the app already uses Vercel Cron (`vercel.json` `crons`) or a cron library for this task, remove that schedule once the Oakstack job works, so the task doesn't run twice.

## Webhook relay (hook)

Use it in front of the app's existing webhook route. The app code barely changes.

```ts
const endpoint = await oakstack.hook.endpoints.create(
  { name: "Stripe", forwardUrl: "https://yourapp.com/api/webhooks/stripe" },
  { idempotencyKey: "stripe-endpoint-v1" },
);
console.log(endpoint.url); // https://oakstack.dev/h/...
```

Then tell the user to replace the webhook URL in the sender's dashboard (Stripe: Developers → Webhooks) with `endpoint.url`. The app keeps verifying the sender's signature (for example `stripe.webhooks.constructEvent`) exactly as before: Oakstack forwards the original headers and the exact body bytes. Read the raw body once with `await req.arrayBuffer()` or `await req.text()` before any JSON parsing. Optionally also verify `Oakstack-Signature` with `endpoint.signingSecret`.

The handler must be safe to receive an event twice (delivery is at least once). Dedupe on the sender's event id or the `oakstack-event-id` header.

Debugging: `await oakstack.hook.endpoints.events(endpoint.id, { status: "failed" })`, then `await oakstack.hook.events.get(id)` shows each attempt's status code and response. After fixing the handler, run `await oakstack.hook.events.replay(id)`.

## Errors

The SDK throws `OakstackError` with `errorName` and a `message` that says what to fix.

- `invalid_request`: fix the field named in the message (bad cron, unknown time zone, localhost or `http://` URL).
- `usage_limit_reached`: the plan is full (Free allows 3 jobs and 1,000 webhook events a month). Don't retry. Tell the user to upgrade at https://oakstack.dev/dashboard/billing, or to delete or pause jobs they don't need.
- `invalid_api_key`: the key is wrong or revoked; ask the user to check `OAKSTACK_API_KEY`.
- Network errors, 5xx responses, and rate limits are retried automatically by the SDK.

## Don't

- Don't point jobs or endpoints at localhost, private IPs, or `http://` URLs. They're rejected.
- Don't put the API key or signing secrets in client-side code or commit them.
- Don't verify signatures against re-serialized JSON. Verify the raw body.
- Don't create a job or endpoint again on every deploy. Create once with an `idempotencyKey`, or check `list()` first.
