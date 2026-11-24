---
name: secuveri-workflows
description: Design, create or change a secuveri verification workflow (which checks a person goes through, their options and price). Use when someone asks for an age check, KYC, onboarding, AML or re-verification flow in secuveri, or wants an existing workflow adjusted.
---

# secuveri workflows

A **workflow** is the ordered list of steps a person goes through (for example `identity-document`, `liveness-check`, `face-match`, `age-estimation`, `aml-screening`) with per-step options, a minimum age and outcome rules. Sessions run a **published** workflow; a sandbox (`sk_test_`) key runs sandbox workflows, a live key production ones.

This skill works through the secuveri MCP server. If its tools are missing, ask the person to connect it (`claude mcp add --transport http secuveri https://mcp.secuveri.com/mcp`) or to build the workflow in the dashboard under Workflow.

## Steps

1. **Pin the requirement.** Establish what is being proven (age only, identity, identity for a regulated onboarding), the minimum age if any, the countries and documents expected, and how much friction the use case tolerates. Done when you can state it in one sentence, for example "buyers of spirits must be 18+, Swiss and EU customers, low friction".
2. **Read the catalog.** Call `get_workflow_catalog` for the steps, options and prices that exist now, and `list_workflows` for what the organization already has. Reuse or adjust an existing workflow before creating a new one. Done when every step you plan to use appears in the catalog with its price.
3. **Draft.** Pick the fewest steps that prove the requirement:
   - Age gate, low friction: `liveness-check` + `age-estimation`, with the ID document only when the estimate is not clear. Strict or regulated age gate: `identity-document` + `liveness-check` + `face-match` and `minimum_age`.
   - Identity (KYC): `identity-document` + `liveness-check` + `face-match`; add `aml-screening` where the business must screen sanctions and PEP lists, and `nfc-required` where chip reading is wanted.
   - Swiss FINMA online identification has its own profile and rules: read it with `search_docs` before drafting.

   Save it with `create_workflow` (a draft changes nothing live). Done when the draft exists and you have shown the person its steps, options and price per verification.
4. **Publish only on a yes.** `publish_workflow`, and `update_workflow` on a published workflow, change the checks people go through. Ask the person responsible, naming the workflow, its environment (sandbox or production) and what changes, and publish only after an explicit yes. Done when the workflow is published and the person confirmed it.
5. **Test it.** Sessions run published workflows only. With the sandbox connection, create a link with `create_verification_link` and the `workflow_id`, optionally with a `sandbox_scenario` such as `approved` or `declined_face_mismatch`, then read the outcome with `get_verification`. Done when the expected outcome comes back and the person has the workflow id for their integration (see the `secuveri-integration` skill).

## Notes

- Prices come from the catalog and the account, never from memory. Production sessions are charged per step from the prepaid balance; the first 50 each month are free.
- `screen_person_aml` is a separate, charged one-off screening: call it only on an explicit request.
