---
name: secuveri-integration
description: Build or review code that uses secuveri identity or age verification. Use when adding secuveri to an app or shop (sessions, the hosted or embedded flow, signed results, webhooks), when an age check or KYC step must gate an order or account, or when reviewing an existing secuveri integration.
---

# secuveri integration

secuveri verifies a person (ID document, liveness, face match, age, AML) on its own pages. Your server starts a **session**, the person completes the **flow**, and your server trusts only the **signed result**, never the browser.

API base: `https://api-verify.secuveri.com`, key in the `x-api-key` header. Keys: `sk_test_` runs sandbox workflows, `sk_live_` production ones. Docs: https://secuveri.com/docs.

When the secuveri MCP server is connected, use it for live facts instead of guessing: `get_account_info` (organization id, environment), `list_workflows` / `get_workflow` (the workflow id and what it checks), `search_docs` and `get_code_sample` (current API details), `create_verification_link` (a quick sandbox test).

## Steps

1. **Map the gate.** Find the exact point that must wait for the verification (placing an order, opening an account, releasing a payout) and the record it belongs to (order, user, application). Done when you can name the function that must refuse without an approved result, and the id that ties the session to that record.
2. **Configure.** Server-only environment variables: `SECUVERI_API_KEY`, `SECUVERI_ORGANIZATION_ID`, `SECUVERI_WORKFLOW_ID`, `SECUVERI_WEBHOOK_SECRET` (`whsec_…`). Done when none of them can reach client code or logs.
3. **Create the session on the server.** `POST /v1/sessions` with `workflow_id`, `vendor_data` = your record id, and an `Idempotency-Key` derived from that id (for example `age-check-<orderId>`). Store the returned `verification_id` on the record. Done when retrying the same request returns the same session (`reused: true`) instead of a second charge.
4. **Show the flow.** Redirect to the session `url`, or embed it with `@secuveri/web` (`Secuveri.start({ sessionUrl, mode: "modal" })`). Embedding needs the site's origin under Settings → Branding → Websites that may embed the flow. The browser's `onComplete` only means "go and check": call your server, which does step 5. Done when nothing in the browser can mark the record as verified.
5. **Settle from the signed result.** `GET /v1/verifications/{id}/result` returns `{ "result": "<JWS>" }` (409 while unfinished). Verify it, then apply the **gate rules** below. Run the same settle function from the browser callback and from the webhook, whichever comes first, and make it idempotent. Done when every gate rule is enforced in code and covered by a test.
6. **Receive webhooks.** Verify `secuveri-signature` over the raw body before parsing, answer 2xx within 10 seconds, drop duplicates by event `id`, then call the settle function. Done when an unsigned or tampered request gets 400 and a duplicate changes nothing.
7. **Test in the sandbox.** With a `sk_test_` key, send `sandbox_scenario` on session create to finish at once: `approved`, `declined_tampered`, `declined_expired`, `declined_unreadable`, `declined_face_mismatch`, `review_liveness`, `review_aml`, `review_document`, `check_error`. Done when approved, declined and review each end in the right state, through both the browser path and the webhook.

## Gate rules

Approve only when **all** hold for the verified result's `verification` object:

- The signature is valid: ES256, key chosen by `kid` from `https://api-verify.secuveri.com/.well-known/jwks.json` (cache a few minutes, refetch once on an unknown `kid`), and `aud` equals your organization id.
- `vendor_data` equals this record's id, and `workflow_id` equals the workflow you started. A valid result for someone else's cart is a replay.
- `status` is `COMPLETE` and `verdict` is `APPROVED`. `REVIEW_REQUIRED` is pending, never approved: a reviewer decides later and a new webhook follows.
- For age gates, `age_check` covers the required age: `estimation` needs `passed` and `pass_age` ≥ the age; `document` needs `minimum_age` ≥ the age; `swiss_eid` and `altid` need `passed` and an `age_over_NN` attribute with NN ≥ the age. A missing `age_check` means the workflow has no age step: treat it as not covered.
- In production, a result carrying `sandbox_scenario` is refused.

Keep the JWS on the record as evidence. It holds the outcome and identifiers, never the person's documents, face or name.

## Webhook signature

`secuveri-signature: t=<unix seconds>,v1=<hex>`. The hex is HMAC-SHA256 of `<t>.<raw body>` with the endpoint's `whsec_` secret. Compute it over the bytes as received, compare in constant time, refuse timestamps older than 5 minutes, and accept any of several `v1` values (secret rotation). The body carries identifiers and the outcome only; read the signed result for the decision.

## Language references

- Node.js / TypeScript (`@secuveri/node`, `@secuveri/web`): [references/node.md](references/node.md)
- Any other language over plain HTTP (Python example): [references/http.md](references/http.md)

## Reviewing an existing integration

Check it against every step's completion criterion and every gate rule, and report each one as met or missing with the file and line. The usual gaps: approval from the browser callback, no `vendor_data` / `workflow_id` comparison, `REVIEW_REQUIRED` treated as approved, the webhook parsed before its signature is checked, and sandbox results accepted in production.
