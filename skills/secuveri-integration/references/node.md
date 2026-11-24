# Node.js / TypeScript

`npm install @secuveri/node` (server) and `@secuveri/web` (browser, optional). `@secuveri/node` runs on Node 20+, Bun, Deno and edge runtimes (Web Crypto). Call `get_code_sample` on the MCP server for the current version of these snippets.

## Client (server only)

```ts
import { Secuveri } from "@secuveri/node";

export const secuveri = new Secuveri({
  apiKey: process.env.SECUVERI_API_KEY!,
  organizationId: process.env.SECUVERI_ORGANIZATION_ID!, // checked against the result's aud
});
```

`secuveri.sandbox` is true for a `sk_test_` key.

## Start a session

```ts
const session = await secuveri.createSession(
  {
    workflow_id: process.env.SECUVERI_WORKFLOW_ID,
    vendor_data: order.id,
    metadata: { order_id: order.id }, // returned in webhooks and results; no personal data
    ...(secuveri.sandbox && scenario ? { sandbox_scenario: scenario } : {}),
  },
  { idempotencyKey: `age-check-${order.id}` }
);
await saveVerificationId(order.id, session.verification_id);
return { sessionUrl: session.url }; // to the browser
```

## Embed the flow (browser)

```ts
import { Secuveri } from "@secuveri/web";

Secuveri.start({
  sessionUrl,
  mode: "modal",
  onComplete: () => fetch(`/api/orders/${orderId}/settle`, { method: "POST" }), // the server decides
  onError: (error) => showError(error.message), // load_timeout: origin not allowed under Branding
});
```

## Settle (browser callback and webhook both call this)

```ts
import { SecuveriApiError, type ResultPayload } from "@secuveri/node";

export async function settle(order: Order) {
  if (order.status !== "awaiting_verification") return order; // idempotent
  let result;
  try {
    result = await secuveri.getResult(order.verificationId); // fetches and verifies the JWS
  } catch (error) {
    if (error instanceof SecuveriApiError && error.status === 409) return order; // not finished yet
    throw error;
  }
  const decision = decide(result, order);
  return saveDecision(order.id, decision, result.jws); // keep the JWS as evidence
}

function decide(result: ResultPayload, order: Order) {
  const v = result.verification;
  if (v.vendor_data !== order.id || v.workflow_id !== process.env.SECUVERI_WORKFLOW_ID) return "rejected";
  if (!secuveri.sandbox && v.sandbox_scenario !== undefined) return "rejected";
  if (v.status === "REVIEW_REQUIRED") return "pending";
  if (v.status !== "COMPLETE" || v.verdict !== "APPROVED") return "rejected";
  return coversAge(v.age_check, order.minimumAge) ? "approved" : "rejected";
}

function coversAge(age: ResultPayload["verification"]["age_check"], minimum: number) {
  if (!age) return false;
  if (age.method === "estimation") return age.passed && age.pass_age >= minimum;
  if (age.method === "document") return age.minimum_age !== null && age.minimum_age >= minimum;
  return age.passed && Number(String(age.attribute).replace("age_over_", "")) >= minimum;
}
```

For a check without an age requirement, drop `coversAge`.

## Webhook route

```ts
import { SecuveriSignatureError } from "@secuveri/node";

export async function POST(request: Request) {
  const rawBody = await request.text(); // raw bytes, before any JSON parsing
  let event;
  try {
    event = await secuveri.verifyWebhook(
      rawBody,
      { signature: request.headers.get("secuveri-signature"), jws: request.headers.get("secuveri-jws") },
      process.env.SECUVERI_WEBHOOK_SECRET!
    );
  } catch (error) {
    if (error instanceof SecuveriSignatureError) return new Response(null, { status: 400 });
    throw error;
  }
  if (await alreadyHandled(event.id)) return new Response(null, { status: 204 });
  const order = event.data.verification_id ? await findOrderByVerification(event.data.verification_id) : null;
  if (order) await settle(order);
  return new Response(null, { status: 204 });
}
```

Frameworks that parse JSON first (Express `express.json()`, Next.js pages API with `bodyParser`) must keep the raw body for this route: `express.raw({ type: "application/json" })`, or `export const config = { api: { bodyParser: false } }`.
