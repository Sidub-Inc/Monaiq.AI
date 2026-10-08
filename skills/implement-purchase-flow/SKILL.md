---
name: implement-purchase-flow
description: "Use when: adding an in-app license purchase flow, buy button, Stripe checkout session, checkout result handling, credential persistence, or post-purchase SDK refresh."
agent: monaiq
auto-invoke:
  - "User wants to add in-app purchases or checkout to their application"
  - "User wants to integrate Stripe checkout for license purchasing"
  - "User asks how to embed a buy button or purchase flow"
tags: [sdk, checkout, purchase, licensing, stripe]
category: integration
allowed-tools: [Read, Write, Edit, Grep, Glob, Bash, profile, offering, feature_offering, implement_purchase_flow, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__offering, mcp__plugin_monaiq_monaiq__feature_offering, mcp__plugin_monaiq_monaiq__implement_purchase_flow, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "platform (dotnet|dotnet/blazor-server|react|react/vite|react/nextjs)"
tier: 3
invoked-by: [implement-licensing, manage-catalog]
---

<objective>
Add an in-app license purchase flow tied to published offerings, reliable customer identity, credential persistence, post-purchase state refresh, and host-native recovery states. This is a leaf skill; it completes checkout integration rather than handing off to another implementation skill.
</objective>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — checkout architecture and credential persistence require hard checkpoints).
</monaiq-agent-handoff>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only checkout, result handling, credential persistence, post-purchase refresh, and purchase validation decisions.
</execution_context>

<input-output-contract>
Input from `implement-licensing` or `manage-catalog`: optional `sdkConfig`, optional `catalogSpec`, route/journal context, and target offering evidence. Direct invocation must verify SDK integration and discover sellable offerings before checkout work.

Output: `checkoutImpl: { architecture: "buyer-bearer" | "server-initiated" | "hosted-page" | "client-only", offeringCodes: [], checkoutRoute: string, successHandler: string }` plus validation status.
</input-output-contract>

<tool-first-authority>
implement_purchase_flow is authoritative for checkout, credential persistence, post-purchase refresh, and purchase-result handling. Use `implement_purchase_flow` and `fetch_step_resources` as the normal implementation guidance paths before code/config/business-logic changes.

Fetch canonical platform SDK resources before checkout code/config/business-logic edits: `monaiq://platforms/api-surface/{platform}`, `monaiq://platforms/manifest/{platform}`, `monaiq://sdk/{stack}/setup`, `monaiq://config/endpoints`, and `monaiq://docs/anti-patterns/{platform}`. Treat the source skill, `implement_purchase_flow` response, canonical resources, and checkpoint/journal evidence as one required evidence set before checkout edits are allowed or a checklist gate is marked complete.

manual HTTP-client checkout implementation is a plugin guidance defect when SDK/tool guidance exists or should exist. Do not ask the user to approve a hand-rolled `HttpClient`, `fetch`, `axios`, or direct gateway checkout workaround as the normal path.

SDK reverse-engineering is a plugin guidance defect. Do not inspect DLLs. Do not inspect XML documentation. Do not inspect NuGet package caches. Do not inspect generated binaries to discover SDK signatures, checkout request fields, credential persistence semantics, endpoint values, or post-purchase behavior.

If Monaiq tool or resource guidance is missing, contradictory, or insufficient, stop before code/config/business-logic changes and record a plugin guidance defect with monaiq_journal record_error. Include the missing or contradictory fact/resource, expected SDK/tool authority, affected platform/skill/checklist gate, blocked checklist gate, blocked next action, and a no-secret defect summary. Mark the purchase-flow checklist gate blocked rather than reverse-engineering or hand-rolling checkout.
</tool-first-authority>

<checkpoint-workflow-directive>
For `CHECKPOINT-CHECKOUT-ARCHITECTURE`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE`, and any other hard checkpoint, follow `_shared/workflows/checkpoint.md` exactly. Use `checkpoint` in `monaiq_journal save_checkpoint` payloads, record the user's `result`, apply returned `.monaiq/*` file operations, and verify the checkpoint file exists before purchase-flow edits continue.
</checkpoint-workflow-directive>

<workflow>
1. Run `_shared/workflows/startup.md` for `implement-purchase-flow`.
2. **Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **purchase flow architecture + credential persistence**. Evidence sources, in priority order: existing routes / server-API boundaries / auth-session shape / purchase UI entry points in `targetProject`, SDK integration state, published offerings, credential handling policy from journal, route packet, `monaiq://sdk/{stack}/setup`, `monaiq://config/endpoints`, `monaiq://docs/anti-patterns/{platform}`. Per-stack code shapes live in `monaiq://sdk/{stack}/setup` and the `implement_purchase_flow` server response — do not embed framework-specific code in this skill source.
3. Read the master journey checklist from `.monaiq/STATE.md`; this skill owns only the purchase flow when applicable gate. Establish prerequisites: SDK integrated, active session, profile credentials available server-side, published offerings available, and resources fetched. Missing SDK routes to `implement-licensing`; missing offerings route to `manage-catalog`.
3. Detect existing checkout implementation and purchase UI entry points before proposing new routes/components.
4. Use evidence before asking a new question. Infer checkout architecture from existing routes, server/API boundaries, auth/session shape, purchase UI entry points, SDK state, offering facts, credential handling policy, and journal decisions. You must confirm inferred decisions in the next existing checkpoint, especially `CHECKPOINT-CHECKOUT-ARCHITECTURE`, with labeled assumptions for checkout architecture, feature path, credential handling, and next steps. Do not add a new checkpoint name solely for evidence inference.
5. Present the plan using `_shared/response-patterns.md` "Evidence Backing" plus selected offerings, architecture, credential persistence policy, and validation plan.
6. Call `implement_purchase_flow` with `startStep=all` when platform, offering availability, required resources, credential handling, and checkout architecture are known; use numbered steps only when step-by-step review improves safety.
7. Stop at `CHECKPOINT-CHECKOUT-ARCHITECTURE`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE`, or `CHECKPOINT-PRE-APIKEY-EXPOSURE-RISK` before checkout routes, success handlers, provider behavior, post-purchase refresh, credential persistence, or frontend/API-key exposure decisions. Do not journal ApiKey or EncodedCredential values.
8. Apply checkout, success, persistence, and UI changes using the authoritative tool/resource guidance only. If guidance is missing, contradictory, or insufficient, stop and record a plugin guidance defect.
9. Validate checkout session creation, result retrieval, credential storage, state refresh, and feature checks. On checkout/build/validation failure, call `monaiq_journal record_validation_failure` before remediation.
10. Coalesce changed paths, validation proof, purchase-flow checklist progress, and `checkoutImpl` handoff through `_shared/workflows/completion.md`. Call `update_checklist_progress` for purchase flow when applicable only after source skill, relevant MCP tool, canonical resources, and checkpoint/journal evidence prove the gate is complete before marking the checklist gate complete. The purchase-flow checklist item cannot be marked complete without SDK/tool evidence: implement_purchase_flow evidence, canonical resource evidence, and checkpoint/journal evidence must all be present. If any evidence is missing, record a blocked checklist gate through the plugin guidance defect path. Save `CHECKPOINT-SKILL-COMPLETE` with `proofOfDone` only when useful, apply returned operations using the **File Operation Application Protocol** in `_shared/protocols.md`, call `skill_completed` once, and hand off persisted `checkoutImpl` plus `validationProof`.
</workflow>

<experience-contract>
## Purchase Experience Contract

Design purchase UI around the host app's job, not a universal Monaiq screen. A full license page or purchase page is appropriate when the user is comparing plans, managing billing, or intentionally reviewing account state; a compact feature-level upsell is appropriate beside a locked action when the user already understands the task and needs the shortest trustworthy upgrade route. In both cases, keep the experience host-native, calm, task-focused, and persuasive without turning the application into a marketing page.

Every purchase surface must explain the current license state, the target offering, included features, limits, price, and fit in business-readable language. Offer comparison surfaces should help the user choose confidently by showing plan differences, included features, quota or usage limits, price, and who each plan is for without leading with technical licensing vocabulary. Upgrade copy must explain the current-to-target upgrade path and what value is gained.

Before direct checkout, require reliable email, customer identity, and offering context. Customer email is the purchase identity used for Stripe pre-fill, license delivery, and payment follow-up; if the app cannot provide a reliable email, pause for a host-native question or route to identity/profile work before checkout. Direct checkout from a compact upsell is allowed only when the customer, reliable email, target offering, success path, and recovery path are known.

After checkout completes, refresh license state, read the current entitlement snapshot, show newly unlocked features or the current entitlement state, and provide the next useful action. Cover loading, available, locked, near-limit, over-limit, expired, success, misconfigured, and recovery states so the user never lands in a vague purchased-but-still-blocked experience.

Placement is a product decision. Prefer account/settings/billing pages for broad license management, feature-adjacent surfaces for contextual locked-feature upsells, and admin/tenant surfaces for buyer-managed purchases, but verify against the host app's navigation, component system, accessibility, responsive behavior, and existing visual language. Do not impose a Monaiq visual standard.

Before UI code changes, present `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT` with a summary naming affected pages, components, flows, and user-visible impact. Preserve `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE` before writing purchased credentials and record the approval result without ApiKey or EncodedCredential values.

</experience-contract>

<reference>
## State Routing Reference

- SDK not integrated -> route to `implement-licensing` first.
- No offerings exist -> route to `manage-catalog` to create offerings.
- Offerings exist but all are Draft -> warn that offerings need to be Public for checkout.
- Existing checkout code found -> show current implementation and offer modifications.

## Discovery Reference

Determine your application's checkout architecture and identify the offerings to sell.

**Architecture decision (platform-neutral):**

| Architecture | Flow | Recommendation |
|-------------|------|----------------|
| Distributed app the buyer runs (desktop, mobile, CLI, installed shell) | Buyer-bearer purchase — the buyer signs in and buys with their own bearer token | Recommended — no seller secret ships with the app; catalog read live, anonymously. |
| Has backend (API, BFF, SSR) | Server-Initiated Checkout (key path, account API key) | Recommended for server-side code — the key and correlation ID stay server-side. |
| Browser SPA with no backend | Hosted marketplace page | Link the buyer to the seller's storefront; a browser must never hold the account API key. |

**CorrelationId** is a tracking identifier (called `CorrelationId`) that links the purchase back to the buyer. It is an opaque string your application provides that round-trips through the entire checkout flow:

1. Your app sends `CorrelationId` with the checkout request.
2. Paid: Stripe Checkout completes, the webhook fires, the license and its seat(s) are created. Free: the buyer-bearer path mints the license at once; the key path first sends the buyer to the claim link.
3. The checkout-result call returns `CorrelationId`, the license ID and its seat IDs — plus the license code when one seat was bought.
4. Your app matches `CorrelationId` back to the user and stores the credential.

Use a stable, unique identifier — user ID, tenant ID, or a composite key.

**Browse offerings** using the `offering` tool to see what is available for purchase. Free, trial, and zero-dollar offerings are fully supported — the API handles them without a Stripe redirect.

For the authoritative offering entity shape and checkout-request field names:

Fetch `monaiq://domain/model` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Step 2: Backend Integration

Create a checkout by calling the SDK's purchase client (buyer-bearer path) or checkout service (key path). The request carries the offering ID, the reseller identifier (`IssuerClientId`), the correlation ID, an optional seat quantity (1 to 100), return URLs and — on the key path only — the customer email. The response carries a redirect URL for a paid offering. For a free / trial offering there is none: on the buyer-bearer path it auto-completes; on the key path it carries a `ClaimUrl` (below).

For the platform-specific checkout-service type, request/response shapes, and invocation pattern:

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For the runtime wiring narrative (where to inject the service, how to configure transport):

Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For the checkout endpoint base URL (do not hardcode — resolve at configuration time):

Fetch `monaiq://config/endpoints` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

**Security notes (platform-neutral):**
- Never expose the `ApiKey` to frontend code or ship it in an app the buyer runs — distributed apps use the buyer-bearer path, browsers proxy through your backend.
- `IssuerClientId` identifies your reseller account — obtained from the `profile` tool. The `ApiKey` (key path) is revealed to owners of the account only.
- Use HTTPS for every success / cancel URL.

**Key path, free offering, typed address:** the create response may carry a `ClaimUrl`. When it
does, nothing has been provisioned yet — the address names the buyer but does not prove them.
Redirect the buyer to the link now, in the same browser; never make them wait for an email (the
same link is emailed only as a fallback for a buyer who closed the tab). Set `ReturnUrl`: they
confirm, prove the address with a one-time code, and land on `ReturnUrl?session_id=…`, where one
result read is enough. Without a `ReturnUrl`, keep polling: the result reads pending until the
buyer confirms, then completed with the credential, or failed with `Claim expired.` once the
seven-day window lapses. Do not add an "awaiting claim" status of your own;
the status vocabulary is unchanged.

## Step 3: Success Handling

After the user completes Stripe Checkout, retrieve the result using the session ID. The result exposes the checkout status, the `CorrelationId` you originally supplied, the license ID, the purchased seat IDs, and the encoded credential string.

Persist the credential against the user identified by `CorrelationId` — this credential is what the licensing SDK uses at runtime.

Purchased `EncodedCredential` values come from checkout-result retrieval or application storage, not the reseller profile. The value belongs to the seat just bought: it carries a seat-scoped runtime token, authorizes and meters that one seat, and the buyer can revoke it from their seat page. A purchase of more than one seat returns no credential at completion — issue one per seat as it is assigned. Store the whole string as issued — never split it, never decode the token inside it, never log it.

For the platform-specific result type and retrieval call pattern:

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For platform-specific persistence patterns and pitfalls (e.g., web storage semantics, secure-string handling):

Fetch `monaiq://platforms/pitfalls/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

### After purchase completes

Once the credential is persisted, **refresh the client and read current state** so the rest of the app
sees the new entitlements without a full reload:

- **React / Node:** `await client.refresh({ ...client.getConfig(), encodedCredential }); const state = await client.getState();`
- **.NET:** `var state = await provider.GetState(serviceReference, context);`

Wire these two calls into whatever success-callback shape your app already uses — the Stripe webhook
handler, a redux thunk, an API-route response, a Blazor event handler. The SDK exposes no
`onPurchaseComplete` hook; this is the canonical two-line idiom.

For the platform-specific signatures:

Fetch `monaiq://platforms/manifest/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Step 4: Context Provider Wiring

Connect the purchased credential to the licensing SDK so feature checks work at runtime.

**Decision (platform-neutral):**

| Credential scope | Wiring pattern |
|------------------|----------------|
| User-managed (multi-tenant, per-user) | Implement a custom licensing-context provider that resolves the credential from your user storage; register it in place of the default provider. |
| Tenant with seats (the app has its own accounts and tenants) | License the tenant, not the user. A tenant holds a license of one or more seats; the backend issues one credential per seat and stores it against the tenant. The context provider returns the credential of the seat the application assigns to the request (the plain case: a seat for each user); the SDK never chooses a seat. End users never hold a license code. Step 4's `tenantSeats` content carries the recipe: checkout per tenant with `CorrelationId`, one credential per seat, seat assignment as the application's own rule, associating a purchase made elsewhere by `LicenseId` (linked to one tenant unless the app shares purchases on purpose), upgrade by association, sync on signals rather than a timer, machines on one seat. Guide: `https://docs.monaiq.com/integrate/tenant-licensing/`. |
| Configuration-based (single-organization license) | Store the encoded credential in your app configuration and restart; the default configuration-driven provider handles the rest. |

For the platform-specific provider interface, method signatures, and registration call:

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For the SDK wiring narrative, including where the provider slots into the composition root:

Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For platform-specific pitfalls (provider lifetime, re-resolution on credential change, lazy initialization):

Fetch `monaiq://platforms/pitfalls/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Verification

1. Create a checkout session with a test offering.
2. Complete the Stripe Checkout flow (or verify a free / trial offering auto-completes).
3. Retrieve the checkout result and confirm the encoded credential is returned.
4. Store the credential and verify the runtime authorization call returns a valid result.
5. Verify feature checks work with the new license.

## Related Tools

- `implement_purchase_flow` — Interactive step-by-step checkout integration (call `startStep=1` through `startStep=4` consecutively).
- `offering` — Browse available product offerings.
- `feature_offering` — View feature assignments for an offering.
- `profile` — Retrieve `IssuerClientId`, and `ApiKey` for an owner of the account (key path only).

## Related Resources

- `monaiq://platforms/api-surface/dotnet` — .NET checkout service, request/result types, provider interface.
- `monaiq://platforms/api-surface/react` — React client methods, provider component, context-hook signatures.
- `monaiq://platforms/manifest/dotnet` — Full machine-readable .NET platform manifest, including post-purchase state patterns.
- `monaiq://platforms/manifest/react` — Full machine-readable React platform manifest, including post-purchase refresh patterns.
- `monaiq://platforms/pitfalls/dotnet` — .NET pitfalls (DI lifetime, credential persistence).
- `monaiq://platforms/pitfalls/react` — React pitfalls (provider remount, storage semantics).
- `monaiq://docs/anti-patterns/dotnet` — .NET checkout and license-handling anti-patterns.
- `monaiq://docs/anti-patterns/react` — React checkout and license-handling anti-patterns.
- `monaiq://sdk/dotnet/setup` — .NET SDK wiring narrative.
- `monaiq://sdk/react/setup` — React SDK wiring narrative.
- `monaiq://config/endpoints` — Checkout/licensing endpoint base URLs.
- `monaiq://domain/model` — Offering, license, and checkout-session entity shapes.

</reference>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Checkout session creation fails | Session-create call returns error | Verify `IssuerClientId` (and, on the key path, the `ApiKey` — an owner re-fetches it from `profile`). Confirm the offering's Status is Public or Private and the seller holds a Monaiq plan (`profile` step 1 `platformPlan`). |
| Stripe redirect fails | User sees Stripe error page | Verify success and cancel URLs use HTTPS and include the session-ID placeholder expected by the checkout service (see `monaiq://platforms/api-surface/{platform}`). Confirm the offering's `BaseRate` and `Currency` are valid. |
| Checkout-result retrieval returns no credential | Result poll returns an empty encoded-credential value | The checkout may not be complete yet — retry after a short delay; a key-path claim stays pending until the buyer accepts. A completed purchase of more than one seat carries no credential by design — issue one per seat from its seat IDs. If persistent, verify the session ID is correct. |
| Credential storage fails | License purchased but credential lost | Re-call the checkout-result retrieval with the same session ID; results are persistent server-side. |
| Context provider wiring fails | Feature checks return null after purchase | Verify the custom context provider returns the stored credential. Check DI / composition-root registration against `monaiq://sdk/{stack}/setup`. |

A checkout create is idempotent by its request ID: resending the same request ID replays the existing checkout, while a new request ID starts a new one. Completed purchases are recorded server-side and can be recovered.
</error-recovery>

<success_criteria>
- Checkout creation works — returns a Stripe URL (paid), nothing to redirect to on the buyer-bearer free path, or a claim URL on the key-path free path.
- Checkout-result retrieval returns the correlation ID, license ID and seat IDs after the purchase completes, and the encoded credential for a one-seat purchase.
- Credential is stored and the runtime authorization call returns a valid result using the purchased license.
- Feature checks work at runtime with the purchased credential.
- `ApiKey` is not exposed to frontend code in production.
</success_criteria>
