---
name: profile-onboarding
description: "Use when: viewing Monaiq seller profile status, payouts status, retrieving seller ApiKey (owners only) and IssuerClientId, reviewing terms and privacy documents, setting billing rules, or registering the customer sign-in that skips the emailed code on a free claim."
agent: monaiq
auto-invoke:
  - "User wants to view their seller profile or retrieve SDK credentials"
  - "User wants to review terms of service before accepting"
  - "User asks about their onboarding status or account details"
tags: [profile, onboarding, credentials, terms, payouts]
category: onboarding
allowed-tools: [profile, account, monaiq_journal, fetch_step_resources, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__account, mcp__plugin_monaiq_monaiq__monaiq_journal, mcp__plugin_monaiq_monaiq__fetch_step_resources]
tier: 3
invoked-by: [getting-started]
---

<objective>
Review seller profile status, retrieve seller checkout credentials when requested, and present terms/privacy documents without leaking secret-bearing values. Profile acceptance is a checkpointed tool action, not automatic background work.
</objective>

<input-context>
Receives from getting-started:
- detectedState (optional): { profileComplete, resellerEnabled } — skip viewing if state is known

If invoked without upstream context, checks profile state directly.
</input-context>

<output-context>
Provides to downstream skills:
- profileData: { issuerClientId, profileStatus, resellerStatus, apiKeyAvailable, platformStanding, payoutsState, canSell }

Used by implement-purchase-flow for checkout configuration. Purchased EncodedCredential values come from checkout results, not the seller profile.
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
3. Call `profile` to detect `ProfileStatus`, `ResellerStatus`, `IssuerClientId`, `platformPlan`, `payouts`, `canSell` and `notices`. (Authentication is handled by the MCP client's OAuth flow; on `AuthError`, ask the user to complete the sign-in prompt and retry.)
3a. Read `platformPlan.standing` from the same response — it is the commercial half of "can this account sell", and the status fields do not answer it. Do not skip it because the terms are accepted.
3b. Read `payouts` and `notices` from the same response — the money half of "can this account sell". Tell the user the `payouts.line`, what is in `payouts.due` and by `payouts.deadline`, and send them to `payouts.portalUrl`.
3. Present status in business-readable terms: what the account can do now, what is blocked, and which next action unblocks catalog, checkout, or implementation work.
4. Retrieve seller credentials only when needed for catalog/checkout setup or explicitly requested. Never write raw `ApiKey`, `EncodedCredential`, `.env`, user-secrets, or secret-bearing values into prompts, `.monaiq`, summaries, docs, or generated plugin output.
5. If the user wants to accept terms, stop at `CHECKPOINT-PRE-TERMS-ACCEPTANCE`, follow `_shared/workflows/checkpoint.md`, present the terms/privacy review state, record the user's approval result, then call `profile` step 4 only after approval.
6. Coalesce profile status summaries, checklist progress, and handoff context through `_shared/workflows/completion.md` without secret values, then call `skill_completed` once.
</workflow>

<reference>
## State Routing Reference

- ProfileStatus = NotStarted -> guide full profile setup.
- ProfileStatus = Incomplete -> show missing profile work.
- ResellerStatus = Enabled -> show summary and offer credential retrieval.
- ResellerStatus = Enabled and `platformPlan.standing` is `None` or `Lapsed` -> the account is set up but cannot sell yet. Send the user to `platformPlan.nextStep.url` (the `sellerWizardUri`) and stop; see Platform Standing below.

## View Profile

Call the `profile` tool with `startStep=1` to view the seller profile.

**Key fields:**

| Field | Description |
|-------|-------------|
| `IssuerClientId` | Unique seller identifier — used in SDK configuration and checkout requests |
| `ContactEmail` | Primary contact email for the account |
| `BusinessName` | The name buyers see; required to accept the seller agreement, set by an owner |
| `SupportContact` | Where buyers reach the account for help: an email address or an https link, set by an owner in Settings |
| `LegalName` | Legal business name on file |
| `ProfileStatus` | Account completeness: `NotStarted` → `Incomplete` → `Completed` |
| `ResellerStatus` | Onboarding state: `NotStarted`, then `Enabled` on accepting the seller agreement. There is no approval step |
| `platformPlan` | The account's Monaiq platform standing — what it may SELL (see Platform Standing below) |
| `payouts` | Whether Stripe pays the seller out (see Payouts below) |
| `canSell` | True when terms are accepted, the account holds a plan, and payouts are `Connected` or `Needs attention`; `canSellBlockedBy` lists what is missing (`terms`, `plan`, `payouts`) |
| `notices` | Open account notices: `kind`, `severity`, `line`, `actionPath`, `due`. Relay each to the user; empty when nothing needs them |

**Status meanings:**

- **ProfileStatus = NotStarted** — Account created but no profile information submitted
- **ProfileStatus = Incomplete** — Some profile fields are filled but required information is missing
- **ProfileStatus = Completed** — All required profile information is on file
- **ResellerStatus = NotStarted** — Seller terms not accepted; the seller account is not activated
- **ResellerStatus = Enabled** — The seller account is active: the user can create products and offerings (a completed checkout creates each license). That is not the same as being able to sell — completing a sale additionally requires platform standing and payouts that can pay (see Platform Standing and Payouts below)

`resellerStatusDescription` says "Seller onboarding is complete: seller terms are accepted and the account is active." only when payouts are `Connected` or `Needs attention`. Otherwise it says "Seller terms are accepted and the account is active." followed by the payouts state and what is left. Repeat that sentence; never say onboarding is complete on your own.

The `IssuerClientId` is the primary identifier for the seller account. It is used as the `IssuerClientId` parameter in checkout requests and appears in license metadata.

**Identity is per environment.** One person is a different seller account in each Monaiq environment (production, UAT), and every product, offering and feature id differs between them; an environment reset changes them all. Record the `IssuerClientId` and catalog ids under the environment `profile` names in its `environment` field (`production` or `uat`), never as one value for all environments. Do not copy an id from one environment into another's configuration.

## Platform Standing

`profile` step 1 returns a `platformPlan` block alongside the status fields. Accepted terms and `ResellerStatus = Enabled` establish that the account IS a seller; `platformPlan` establishes whether Monaiq will let it SELL. They are two different facts and an account can hold the first without the second.

**`platformPlan.standing`:**

| Standing | What it means for selling |
|----------|---------------------------|
| `Package` | A paid Monaiq plan (Entry / Growth / Scale) is held. Selling works. |
| `Preview` | The free Preview Monaiq plan is held. Selling works, with the top tier's allowances. |
| `Trial` | A time-limited platform trial is running. Selling works until it ends — `nextStep.daysRemaining` says how long is left. |
| `Lapsed` | A plan was held and is no longer live. New sales are refused; every license already issued keeps validating. |
| `None` | No plan was ever held. New sales are refused. |
| `Unknown` | Standing could not be read this call, with the reason in `platformPlan.error`. Report it; do not treat it as `None`. |

`platformPlan.hasStanding` is the same answer as a boolean, and `platformPlan.allowances` carries what the plan grants (counts, with `Unlimited` spelled out; roadmap items marked `coming`).

**The remedy is always the same, and it is not a tool call.** When standing is `None` or `Lapsed` — or when a `Trial` is close to ending — hand the user `platformPlan.nextStep.url`, which is the `sellerWizardUri` from `monaiq://config/endpoints`. Tell them what it does and stop there.

- **Never attempt to buy a plan through MCP.** No tool acquires one, and none will: acquiring a plan is a commercial act the person performs in the wizard.
- **Never gate your own work on standing.** Catalog CRUD, credential retrieval, SDK guidance, and the marketplace listing all keep working without a plan. No MCP tool refuses on standing; what changes without one is that a BUYER's checkout is refused.
- **Never invent the URL.** Resolve `sellerWizardUri` from `monaiq://config/endpoints`, or use the one `platformPlan.nextStep` handed you.

**What it costs, stated plainly when the user asks:** selling on Monaiq requires a Monaiq plan. The wizard states which plans are open and what each one costs. Do not promise a price, a discount, or a free plan — say what the wizard shows and let the user read it.

## Payouts

`profile` step 1 returns a `payouts` block: `state`, `line`, `due`, `deadline` and `portalUrl`. It is Monaiq's own reading of the seller's Stripe account, in seven words and never Stripe's codes. Say `line` to the user as written.

| `payouts.state` | What it means | canSell |
|-----------------|---------------|---------|
| `Not started` | No Stripe account yet. | false |
| `Setting up` | Stripe is still taking or checking the seller's details. | false |
| `Connected` | Payouts work. | true |
| `Needs attention` | Payouts work, and Stripe needs something by `deadline`; `due` lists it. | true |
| `Paused` | Stripe, or Monaiq, has paused payouts. `due` lists what Stripe wants. | false |
| `Stopped` | Stripe, or Monaiq, has declined the account. | false |
| `Disconnected` | The Stripe account is gone; the seller sets up payouts again. | false |

`due` is empty when nothing is asked; `deadline` is null when Stripe set none.

**The remedy is a page, not a tool call.** For any state but `Connected`, hand the user `payouts.portalUrl` (the `payoutsUri` from `monaiq://config/endpoints`): the Payouts page says what to do and opens Stripe from the user's own signed-in session.

- **Never open, start or fix Stripe yourself.** No tool starts onboarding or calls Stripe, and a Stripe login link is never handed to an agent or sent in a message.
- **Never gate your own work on payouts.** Catalog, credentials and SDK work carry on; what changes is that the seller cannot be paid.
- **Never quote a Stripe code.** If a user asks, the Payouts page has the detail.

`notices` lists what needs the seller right now. Relay each by its `line`, point at `actionPath` on the portal, and mention `due` when it has one.

## Retrieve Credentials

Call the `profile` tool with `startStep=2` to retrieve seller credentials used for catalog and checkout operations. The `ApiKey` is revealed to owners of the account only (as on the portal's credentials page): a member's response carries `apiKey: null` and a `troubleshooting` block naming the rule — tell the member to ask an owner for the key, or to have an owner run `provision_api_key_config`. Use the `account` tool to see which account, and which role, this connection acts as.

**Credential fields:**

| Credential | Purpose | Where Used |
|-----------|---------|------------|
| `ApiKey` | Authenticates SERVER-SIDE checkout API calls; it is the whole ACCOUNT, including what that account sells with | `CreateCheckoutSession` and `GetCheckoutResult` API key parameter |
| `IssuerClientId` | Seller identity for checkout requests | `CheckoutRequest.IssuerClientId` |

EncodedCredential is not a seller profile credential. It belongs to a SEAT of a license: it is produced when an offering is purchased and returned by checkout-result retrieval, or issued on the seat page by the account that bought it. It carries a seat-scoped runtime token, so it authorizes and meters that one seat and reaches nothing else — which is exactly why it, and never the `ApiKey`, is what a distributed application configures.

**Security:** Do not persist the `ApiKey` to disk or commit it to source control. Use environment variables or a secrets manager for production deployments. Never ship it inside an application a buyer runs.
Do not write raw `ApiKey`, `IssuerClientId` plus secret context, `EncodedCredential`, `.env`, or user-secret values into prompts, `.monaiq`, summaries, or generated plugin output.

**How credentials connect to checkout setup:**

- `ApiKey` → Used when calling `ICheckoutService` methods (see the `implement-purchase-flow` skill)
- `IssuerClientId` → Used in `CheckoutRequest` for embedded purchases

## Read Terms

Call the `profile` tool with `startStep=3` to read the Terms of Service and Privacy Policy.

Both documents must be presented to the user before acceptance. Accepting the seller terms also accepts the Seller Agreement, which step 3 returns as `sellerAgreement` when it is published — present it too before the user accepts the seller terms. The terms cover:

- **Terms of Service** — Account registration, use of services, intellectual property, data processing, limitation of liability
- **Privacy Policy** — Data collection, usage, storage practices, user rights under applicable privacy laws

Review both documents carefully before proceeding to accept terms.

## Accepting Terms (Step 4)

**Important:** Accepting terms is NOT part of this skill workflow. It is a **tool action** that modifies state.

To accept terms, call the `profile` tool directly with `startStep=4` and `data={"customerTerms": true, "resellerTerms": true, "businessName": "Kestrel Studio"}`. Each flag is optional — include only the terms the user accepts. Customer terms complete the profile (`ProfileStatus = Completed`); seller terms enable selling (`ResellerStatus = Enabled`), so a seller accepts both. This action records the user's agreement and updates the profile status.

Seller terms need the business name buyers will see. When the account has none, ask the builder for it and send it as `businessName` with `resellerTerms`; without it the step answers `NeedsBusinessName` and writes nothing. Only an owner can set it: a member's acceptance answers `OwnerOnly`. A `businessName` sent when the account already has one is ignored; the name changes in Settings.

## Billing Rules (Step 5)

A seller's rules for the customers they bill live on the account, not on offerings. Call the `profile` tool with `startStep=5` and `data` holding only the settings to change: `suspensionGraceHours` (0 to 720; how long after a failed card payment, or after an invoice's due date, a customer who has not paid keeps access before the license is suspended; 48 by default; 0 suspends at once), `remindBeforeTrialEnds` (emails the customer 2 days before a trial ends; on by default) and `remindBeforeRenewal` (emails the customer 7 days before a subscription billed monthly or less often renews; on by default). Step 5 with no `data` reads the current values. Only an owner can change them; a member's call answers `OwnerOnly`.

## Customer Sign-in (Step 6)

When the seller's app already signs its customers in with OpenID Connect (Microsoft Entra, Google, Auth0 and the like), registering that sign-in lets a customer the app signed in claim a free license or a trial with one confirm instead of an emailed code. The app sends the customer's ID token as `BuyerIdToken` on the checkout (see the `implement-purchase-flow` skill). Paid checkouts go through Stripe and do not use it.

Call the `profile` tool with `startStep=6` and `data={"issuer": "...", "clientId": "..."}` to register or replace the pair, or both `null` to clear it. With no `data` it reads the current pair; step 1 shows it as `customerSignIn` too.

- `issuer` is compared character for character with the token's `iss`, trailing slash included. Take it from a real ID token the app received; never type it from memory. It must be an `https://` URL on a public host name.
- `clientId` is the client id of the seller's own sign-in app, the token's `aud`. It is not the `IssuerClientId`.
- A refusal answers `ValidationError` with the same sentence the portal's Customer sign-in page shows, and writes nothing. Only an owner can change the pair; a member's call answers `OwnerOnly`.

## Related Tools

- OAuth sign-in — handled by the MCP client (prerequisite for all profile operations; no tool call)
- `profile` — The tool that executes each step of this workflow
- `implement_base` — SDK integration (configures where purchased credentials are supplied at runtime)
- `implement_purchase_flow` — Checkout integration (the buyer-bearer path needs only IssuerClientId; the server-side key path also needs the ApiKey from step 2)
- `account` — Which of the user's accounts this connection acts as, and the role held there

## Utility Workflows

- **Credential recovery**: Re-retrieve seller checkout credentials from your profile. Use purchase-flow checkout results or your application storage for purchased `EncodedCredential` values.
- **View terms**: Review Terms of Service and Privacy Policy before accepting.
- **Check seller status**: See current ResellerStatus and what to expect next.
- **Check selling readiness**: Read `canSell`. When it is false, `canSellBlockedBy` names what is missing: for `plan`, hand the user `platformPlan.nextStep.url`; for `payouts`, hand them `payouts.portalUrl`.
- **Billing rules**: Read or change the grace before suspension and the two reminder switches with `profile` step 5.
- **Customer sign-in**: Register, read or clear the issuer and client id behind the one-confirm free claim with `profile` step 6.

</reference>

<success_criteria>
- Profile information is visible including `IssuerClientId`, `ProfileStatus`, `ResellerStatus`, `platformPlan.standing`, `payouts.state` and `canSell`
- When `canSell` is false for `payouts`, the user is told the `payouts.line` and pointed at `payouts.portalUrl`; Stripe is never opened for them and no onboarding is said to be complete
- When `platformPlan.standing` is `None` or `Lapsed`, the user is pointed at the `sellerWizardUri` and no purchase is attempted through MCP
- Seller credentials are retrieved: `IssuerClientId` always, `ApiKey` for an owner (a member is told to ask an owner)
- Terms of Service and Privacy Policy are presented for review
- Step 4 (terms acceptance) is understood as a separate tool action with `startStep=4`
- No raw `ApiKey`, `EncodedCredential`, `.env`, or user-secret values are written into prompts, `.monaiq`, summaries, or generated plugin output
</success_criteria>
