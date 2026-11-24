# secuveri plugin for Claude Code

Identity and age verification with [secuveri](https://secuveri.com) from Claude Code.

- **MCP server** (`https://mcp.secuveri.com/mcp`): create verification links, read outcomes, build and publish workflows, manage block and allow lists, run AML screenings, search the docs. You sign in to your secuveri account and approve what the connection may do.
- **`secuveri-integration` skill**: Claude builds or reviews an integration in your code the safe way: sessions with idempotency keys, the hosted or embedded flow, decisions only from the signed result, webhook signature checks, sandbox tests.
- **`secuveri-workflows` skill**: Claude designs a workflow with you from the live catalog and prices, tests it in the sandbox, and publishes only after you confirm.

## Install

```
/plugin marketplace add secuveri/claude-plugin
/plugin install secuveri@secuveri
```

Then run `/mcp`, pick **secuveri** and sign in. Use a sandbox (`sk_test_`) setup while you build: sandbox sessions are free and can finish at once with a chosen outcome.

## Try it

- "Add a secuveri age check to the checkout, minimum age 18."
- "Review our secuveri webhook handler."
- "Build a KYC workflow with document, liveness, face match and AML screening."

## Links

- Docs: https://secuveri.com/docs
- Support: hello@secuveri.com

## Licence

MIT
