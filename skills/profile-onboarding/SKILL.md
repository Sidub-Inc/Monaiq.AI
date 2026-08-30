---
name: profile-onboarding
description: "Use when: viewing Monaiq reseller profile status, retrieving reseller ApiKey and IssuerClientId, or reviewing terms and privacy documents."
agent: monaiq
auto-invoke:
  - "User wants to view their reseller profile or retrieve SDK credentials"
  - "User wants to review terms of service before accepting"
  - "User asks about their onboarding status or account details"
tags: [profile, onboarding, credentials, terms]
category: onboarding
allowed-tools: [profile, monaiq_journal, fetch_step_resources, mcp__plugin_monaiq_monaiq__mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__monaiq_journal, mcp__plugin_monaiq_monaiq__fetch_step_resources]
tier: 3
invoked-by: [getting-started]
---

<objective>
Review reseller profile status, retrieve reseller checkout credentials when requested, and present terms/privacy documents without leaking secret-bearing values. Profile acceptance is a checkpointed tool action, not automatic background work.
</objective>

<input-context>
Receives from getting-started:
- detectedState (optional): { profileComplete, resellerEnabled } — skip viewing if state is known

If invoked without upstream context, checks profile state directly.
</input-context>

<output-context>
Provides to downstream skills:
- profileData: { issuerClientId, profileStatus, resellerStatus, apiKeyAvailable, platformStanding }

Used by implement-purchase-flow for checkout configuration. Purchased EncodedCredential values come from checkout results, not the reseller profile.
</output-context>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only session/profile status, terms review, credential availability summaries, and checkpointed terms acceptance guidance.
</execution_context>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — terms acceptance is a checkpointed tool action).
</monaiq-agent-handoff>

<workflow>
1. Run `_shared/workflows/startup.md` for `profile-onboarding`.
2. **Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **profile status + terms acceptance**. Evidence sources, in priority order: `profile` (`ProfileStatus`, `ResellerStatus`, `IssuerClientId`), prior journal decisions, route packet. The recommendation is the next forward action that unblocks catalog/checkout/implementation; the host-native question lets the user proceed, view terms/privacy, or pause.
3. Call `profile` to detect `ProfileStatus`, `ResellerStatus`, `IssuerClientId`, and `platformPlan`. (Authentication is handled by the MCP client's OAuth flow; on `AuthError`, ask the user to complete the sign-in prompt and retry.)
3a. Read `platformPlan.standing` from the same response — it is the commercial half of "can this account sell", and the status fields do not answer it. Do not skip it because the terms are accepted.
3. Present status in business-readable terms: what the account can do now, what is blocked, and which next action unblocks catalog, checkout, or implementation work.
4. Retrieve reseller credentials only when needed for catalog/checkout setup or explicitly requested. Never write raw `ApiKey`, `EncodedCredential`, `.env`, user-secrets, or secret-bearing values into prompts, `.monaiq`, summaries, docs, or generated plugin output.
5. If the user wants to accept terms, stop at `CHECKPOINT-PRE-TERMS-ACCEPTANCE`, follow `_shared/workflows/checkpoint.md`, present the terms/privacy review state, record the user's approval result, then call `profile` step 4 only after approval.
6. Coalesce profile status summaries, checklist progress, and handoff context through `_shared/workflows/completion.md` without secret values, then call `skill_completed` once.
</workflow>

<reference>
## State Routing Reference

- ProfileStatus = NotStarted -> guide full profile setup.
- ProfileStatus = Incomplete -> show missing profile work.
- ProfileStatus = Completed and ResellerStatus = Pending -> explain approval wait state.
- ResellerStatus = Enabled -> show summary and offer credential retrieval.
- ResellerStatus = Enabled and `platformPlan.standing` is `None` or `Lapsed` -> the account is set up but cannot sell yet. Send the user to `platformPlan.nextStep.url` (the `sellerWizardUri`) and stop; see Platform Standing below.

## View Profile

Call the `profile` tool with `startStep=1` to view the reseller profile.

**Key fields:**

| Field | Description |
|-------|-------------|
| `IssuerClientId` | Unique reseller identifier — used in SDK configuration and checkout requests |
| `ContactEmail` | Primary contact email for the account |
| `LegalName` | Legal business name on file |
| `ProfileStatus` | Account completeness: `NotStarted` → `Incomplete` → `Completed` |
| `ResellerStatus` | Onboarding state: `NotStarted` → `Pending` → `Enabled` |
| `platformPlan` | The account's Monaiq platform standing — what it may SELL (see Platform Standing below) |

**Status meanings:**

- **ProfileStatus = NotStarted** — Account created but no profile information submitted
- **ProfileStatus = Incomplete** — Some profile fields are filled but required information is missing
- **ProfileStatus = Completed** — All required profile information is on file
- **ResellerStatus = NotStarted** — Reseller terms not accepted; the seller account is not activated
- **ResellerStatus = Pending** — Reseller application submitted, awaiting approval
- **ResellerStatus = Enabled** — The seller account is active: the user can create products, offerings, and issue licenses. That is not the same as being able to sell — completing a sale additionally requires platform standing (see Platform Standing below)

The `IssuerClientId` is the primary identifier for the reseller account. It is used as the `IssuerClientId` parameter in checkout requests and appears in license metadata.

## Platform Standing

`profile` step 1 returns a `platformPlan` block alongside the status fields. Accepted terms and `ResellerStatus = Enabled` establish that the account IS a seller; `platformPlan` establishes whether Monaiq will let it SELL. They are two different facts and an account can hold the first without the second.

**`platformPlan.standing`:**

| Standing | What it means for selling |
|----------|---------------------------|
| `Package` | A paid Monaiq plan (Entry / Growth / Scale) is held. Selling works. |
| `Preview` | The free preview-program package is held. Selling works, with the top tier's allowances. |
| `Trial` | A time-limited platform trial is running. Selling works until it ends — `nextStep.daysRemaining` says how long is left. |
| `Lapsed` | A plan was held and is no longer live. New sales are refused; every license already issued keeps validating. |
| `None` | No plan was ever held. New sales are refused. |
| `Unknown` | Standing could not be read this call, with the reason in `platformPlan.error`. Report it; do not treat it as `None`. |

`platformPlan.hasStanding` is the same answer as a boolean, and `platformPlan.allowances` carries what the plan grants (counts, with `Unlimited` spelled out; roadmap items marked `coming`).

**The remedy is always the same, and it is not a tool call.** When standing is `None` or `Lapsed` — or when a `Trial` is close to ending — hand the user `platformPlan.nextStep.url`, which is the `sellerWizardUri` from `monaiq://config/endpoints`. Tell them what it does and stop there.

- **Never attempt to buy a plan through MCP.** No tool acquires one, and none will: acquiring a plan is a commercial act the person performs in the wizard.
- **Never gate your own work on standing.** Catalog CRUD, credential retrieval, SDK guidance, and the marketplace listing all keep working without a plan. No MCP tool refuses on standing today; what changes without one is that a BUYER's checkout is refused.
- **Never invent the URL.** Resolve `sellerWizardUri` from `monaiq://config/endpoints`, or use the one `platformPlan.nextStep` handed you.

**What it costs, stated plainly when the user asks:** selling on Monaiq requires a Monaiq plan. The wizard states which plans are open and what each one costs. Do not promise a price, a discount, or a free plan — say what the wizard shows and let the user read it.

## Retrieve Credentials

Call the `profile` tool with `startStep=2` to retrieve reseller credentials used for catalog and checkout operations.

**Credential fields:**

| Credential | Purpose | Where Used |
|-----------|---------|------------|
| `ApiKey` | Authenticates checkout API calls | `CreateCheckoutSession` and `GetCheckoutResult` API key parameter |
| `IssuerClientId` | Reseller identity for checkout requests | `CheckoutRequest.IssuerClientId` |

EncodedCredential is not a reseller profile credential. It is produced after an offering is purchased and returned by checkout-result retrieval.

**Security:** Do not persist the `ApiKey` to disk or commit it to source control. Use environment variables or a secrets manager for production deployments.
Do not write raw `ApiKey`, `IssuerClientId` plus secret context, `EncodedCredential`, `.env`, or user-secret values into prompts, `.monaiq`, summaries, or generated plugin output.

**How credentials connect to checkout setup:**

- `ApiKey` → Used when calling `ICheckoutService` methods (see the `implement-purchase-flow` skill)
- `IssuerClientId` → Used in `CheckoutRequest` for embedded purchases

## Read Terms

Call the `profile` tool with `startStep=3` to read the Terms of Service and Privacy Policy.

Both documents must be presented to the user before acceptance. The terms cover:

- **Terms of Service** — Account registration, use of services, intellectual property, data processing, limitation of liability
- **Privacy Policy** — Data collection, usage, storage practices, user rights under applicable privacy laws

Review both documents carefully before proceeding to accept terms.

## Accepting Terms (Step 4)

**Important:** Accepting terms is NOT part of this skill workflow. It is a **tool action** that modifies state.

To accept terms, call the `profile` tool directly with `startStep=4` and `data={"customerTerms": true, "resellerTerms": true}`. Both `customerTerms` and `resellerTerms` must be set to `true` to complete terms acceptance. This action records the user's agreement and updates the profile status.

## Related Tools

- OAuth sign-in — handled by the MCP client (prerequisite for all profile operations; no tool call)
- `profile` — The tool that executes each step of this workflow
- `implement_base` — SDK integration (configures where purchased credentials are supplied at runtime)
- `implement_purchase_flow` — Checkout integration (uses ApiKey and IssuerClientId from step 2)

## Utility Workflows

- **Credential recovery**: Re-retrieve reseller checkout credentials from your profile. Use purchase-flow checkout results or your application storage for purchased `EncodedCredential` values.
- **View terms**: Review Terms of Service and Privacy Policy before accepting.
- **Check approval status**: See current ResellerStatus and what to expect next.
- **Check selling readiness**: Read `platformPlan.standing`; when it is `None` or `Lapsed`, hand the user `platformPlan.nextStep.url`.

</reference>

<success_criteria>
- Profile information is visible including `IssuerClientId`, `ProfileStatus`, `ResellerStatus`, and `platformPlan.standing`
- When `platformPlan.standing` is `None` or `Lapsed`, the user is pointed at the `sellerWizardUri` and no purchase is attempted through MCP
- Reseller credentials (`ApiKey`, `IssuerClientId`) are retrieved successfully
- Terms of Service and Privacy Policy are presented for review
- Step 4 (terms acceptance) is understood as a separate tool action with `startStep=4`
- No raw `ApiKey`, `EncodedCredential`, `.env`, or user-secret values are written into prompts, `.monaiq`, summaries, or generated plugin output
</success_criteria>
