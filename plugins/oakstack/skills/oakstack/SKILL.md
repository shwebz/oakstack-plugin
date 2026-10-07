---
name: oakstack
description: Add scheduled jobs (cron), reliable webhook receiving, PDF generation, transactional email, or reading photos and documents into data, in an app using Oakstack. Use when the user wants something to run on a schedule (nightly cleanup, daily digest, sync every N minutes, reminders), wants Vercel Cron or a cron server replaced with something that retries and logs, or receives webhooks from Stripe, GitHub, Shopify, Clerk, or similar and must not lose events during deploys or outages, needs PDFs (invoices, receipts, proposals, reports, any HTML page as a PDF), sends email (receipts, notifications, password resets, inbound email to the app), or needs to turn photos, screenshots, PDFs, or pasted lists into structured data (product imports, receipts, business cards). Also use when the user mentions Oakstack, OAKSTACK_API_KEY, or the oakstack npm package.
---

# Oakstack

Oakstack is one API key for app infrastructure. Five modules:

- **clock**: Oakstack calls a URL in the app on a cron schedule, retries failures, and logs every run.
- **hook**: a permanent inbound URL for webhooks. Oakstack stores every event, forwards it to the app with the original headers and body, retries for about 11 hours, and can replay any event.
- **print**: HTML or a hosted template (invoice, proposal, report) in, PDF out, rendered with real Chrome and returned with a private download link.

- **lens**: photos, screenshots, PDFs, and pasted text in, structured data out (templates: product-list, receipt, contacts, or your own JSON Schema), with warnings for anything unclear.
- **post**: transactional email from the user's own domain (with PDFs attached in the same call), automatic bounce and complaint suppression, and inbound email delivered to a hook endpoint. Live sending may not be enabled yet: check first (below).

Full docs as markdown: https://oakstack.dev/llms.txt. Fetch https://oakstack.dev/docs/clock.md, https://oakstack.dev/docs/hook.md, or https://oakstack.dev/docs/print.md, or https://oakstack.dev/docs/post.md when you need details.

## Before you start

1. **API key.** Check for `OAKSTACK_API_KEY` in the environment or `.env*` files. If it's missing, ask the user to create one at https://oakstack.dev/dashboard/api-keys and add it to `.env.local` (and to their host's environment variables for production). Never ask them to paste the key into chat, and never commit it.
2. **A public URL (clock and hook only).** Oakstack only calls public `https://` URLs, never localhost. If the app isn't deployed yet, either deploy first and use the production URL, or use a tunnel (`cloudflared tunnel --url http://localhost:3000`) for testing. Say this up front so the user isn't surprised.
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
    process.env.OAKSTACK_JOB_SECRET!,
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

**Step 2: create the job** (once, from a script or with the MCP tool). Make the script safe to re-run: look for an existing job with the same name first (`list()` returns an array). The idempotency key only protects retries within 24 hours. Run it with `npx tsx --env-file=.env.local scripts/setup-oakstack.ts`.

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
  { idempotencyKey: "nightly-cleanup-v1" }, // makes a retried request safe (remembered 24 hours)
);
console.log(job.id, job.signingSecret);
```

**Step 3:** tell the user to add `OAKSTACK_JOB_SECRET=<job.signingSecret>` to the app's environment (local and production). Every job and endpoint has its own secret: with several, give each its own variable named after its purpose (e.g. `OAKSTACK_CLEANUP_SECRET`, `OAKSTACK_WEBHOOK_SECRET`). Then test it: `await oakstack.clock.jobs.run(job.id)`, and check `await oakstack.clock.jobs.runs(job.id)` for `status: "succeeded"`.

Cron cheat sheet: `*/5 * * * *` every 5 minutes, `0 * * * *` hourly, `0 9 * * *` daily at 9:00, `0 9 * * 1-5` weekdays at 9:00, `0 0 1 * *` monthly. Five fields only; there are no seconds. On the Free plan, schedules can run at most every 5 minutes.

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

**Stripe and other senders with timestamped signatures need one change.** Oakstack forwards the sender's original signature header, and its timestamp ages through retries, pauses, and replays. Stripe's `constructEvent` rejects anything older than 5 minutes by default, which would drop exactly the delayed events Oakstack is meant to save. In the webhook route:

1. read the raw body once (`await req.text()`);
2. if an `oakstack-signature` header is present, verify it with `verifySignature(process.env.OAKSTACK_WEBHOOK_SECRET!, ...)`;
3. then call `stripe.webhooks.constructEvent(raw, sig, secret, 60 * 60 * 24 * 30)` (a 30-day tolerance) for relayed requests, and keep the default for direct ones;
4. skip events whose `event.id` was already processed.

The full example is under "Using Oakhook with Stripe" in https://oakstack.dev/docs/hook.md.

Cutover: deploy that route first, then in Stripe **edit the existing endpoint's URL** to `endpoint.url`, which keeps the same `whsec_` secret. A newly added Stripe endpoint gets a new `whsec_`: update `STRIPE_WEBHOOK_SECRET` before disabling the old endpoint.

The handler must be safe to receive an event twice (delivery is at least once). Dedupe on the sender's event id or the `oakstack-event-id` header.

Debugging: `await oakstack.hook.endpoints.events(endpoint.id, { status: "failed" })`, then `await oakstack.hook.events.get(id)` shows each attempt's status code and response. After fixing the handler, run `await oakstack.hook.events.replay(id)`.

## PDFs (print)

No public URL needed: the app calls Oakstack when it needs a PDF. Generate PDFs on the server (an API route, a server action, a job), never in browser code, since it uses the API key.

**Hosted template** (fastest for invoices, proposals, reports). Get the template's example first and change values; the data is checked strictly, and errors name the field.

```ts
import { Oakstack } from "oakstack";
const oakstack = new Oakstack();

const { example } = await oakstack.print.templates.get("invoice"); // also "proposal", "report"
const pdf = await oakstack.print.pdfs.create({
  template: "invoice",
  data: { ...example, invoiceNumber: order.number, to: { name: order.customerName }, items },
  filename: `invoice-${order.number}.pdf`,
  metadata: { orderId: order.id },
});
```

**The app's own HTML** (when it already has a design, or for anything else): build a full HTML document string with inline CSS and send it as `html`. Images must be public `https://` URLs or `data:` URLs (not localhost). JavaScript is off unless `options.javascript: true`. Paper: `options: { format: "A4", landscape: true, margin: "12mm" }`.

**Using the result:** `pdf.url` is a private link valid for an hour; to serve the PDF later, store `pdf.id` and call `oakstack.print.pdfs.get(id)` for a fresh `url`, or redirect the user to it. `await oakstack.print.pdfs.download(pdf.id)` returns the bytes, for attaching to an email or saving to the app's own storage. Oakstack deletes files after 1 day (Free), 7 days (Builder), or 30 days (Studio), so save the bytes if the app needs them long term (an invoice archive).

Check the result visually: download one PDF and open it, or ask the user to. If an image is missing, `pdf.blocked` lists what was refused and why.

## Email (post)

1. **Check what's possible:** `await oakstack.post.status()` (or the `post_status` MCP tool). If `liveSendingEnabled` is false, live email isn't switched on for Oakstack yet: build and test with a test key (`ok_test_`), and tell the user live sending is opening soon. If `membersOnly` is true (Free plan), live email can only go to the workspace's own members until they upgrade.
2. **Test key while building.** An `ok_test_` key checks and logs every email but never delivers it. Simulate outcomes by sending to `bounced@test.oakstack.dev` or `complained@test.oakstack.dev`.
3. **Domain:** `await oakstack.post.domains.create({ domain: "theirdomain.com" })` returns DNS records. Show the user the 3 required CNAMEs (and the recommended MX/TXT/DMARC) to add at their DNS provider; it verifies by itself, usually within an hour.
4. **Send from server code only** (it uses the API key):

```ts
await oakstack.post.emails.send(
  {
    from: "Acme <hello@theirdomain.com>",
    to: user.email,
    subject: "Your receipt",
    html: receiptHtml,
    text: receiptText,
    attachments: [{ filename: `receipt-${order.id}.pdf`, print: { template: "invoice", data } }],
    metadata: { orderId: order.id },
  },
  { idempotencyKey: `receipt-${order.id}` }, // a retry never sends twice
);
```

Hard bounces and spam complaints are suppressed automatically; don't build your own list for that. `oakstack.post.emails.get(id).events` shows delivered, bounced (with the reason), and complained. For inbound email (support inbox, replies), create a hook endpoint pointing at an app route, then `oakstack.post.inbound.create({ name, hookEndpointId })`; the route receives each email as JSON (from, subject, text, html, attachments).

Email errors: `not_allowed` (unverified From domain, or a non-member on Free: the message says which), `sending_paused` (too many bounces or complaints; the user must contact support@oakstack.dev), `email_not_enabled` (use a test key for now).

## Reading documents (lens)

For imports (a photo of products, a pre-order email, a receipt): call it server-side, show the result in an editable review table with the warnings, and save only what the user confirms. Never write extracted data straight into a store or database.

```ts
const result = await oakstack.lens.extract({
  template: "product-list", // or "receipt", "contacts", or schema: { type: "object", ... }
  inputs: [{ data: uploadedFileBytes, filename }, { text: pastedList }],
});
// result.data.items, result.warnings, item.confidence
```

Inputs: up to 10 images (JPEG/PNG/GIF/WebP, 5 MB), PDFs (10 MB, 30 pages), or text; by bytes, base64, or public URL. No HEIC. It takes 5 to 30 seconds; counted in pages (Free 20 a month).

## Errors

The SDK throws `OakstackError` with `errorName` and a `message` that says what to fix.

- `invalid_request`: fix the field named in the message (bad cron, unknown time zone, localhost or `http://` URL).
- `usage_limit_reached`: the plan is full (Free allows 3 jobs, 5 webhook endpoints, 1,000 webhook events a month (replays included), 25 PDFs a month, and 100 email recipients a month). New senders also have a daily email cap (200 recipients a day in their first week), reported as `usage_limit_reached` with the numbers. Don't retry. Tell the user to upgrade at https://oakstack.dev/dashboard/billing, or to delete or pause jobs they don't need.
- `invalid_api_key`: the key is wrong or revoked; ask the user to check `OAKSTACK_API_KEY`.
- Network errors, 5xx responses, and rate limits are retried automatically by the SDK.

## Don't

- Don't point jobs or endpoints at localhost, private IPs, or `http://` URLs. They're rejected.
- Don't put the API key or signing secrets in client-side code or commit them.
- Don't verify signatures against re-serialized JSON. Verify the raw body.
- Don't create a job or endpoint again on every deploy. Create once with an `idempotencyKey`, or check `list()` first.
