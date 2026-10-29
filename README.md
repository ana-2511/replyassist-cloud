# ReplyAssist — automatic replies and 24/7 owner notifications

## Deploy and enable

Upload the extracted root files from this package to the existing private GitHub repository. Netlify expands source.json with unpack.mjs, runs tests, builds dist, and bundles the functions. Keep existing environment variables, encrypted database and OAuth credentials unchanged. Build command: `node unpack.mjs && npm run build`; publish directory: `dist`; Node 24.

After deployment, open the main production URL, verify connections and click **Enable automatic replies + emails**. The first enable sets a new live cutoff to now; it does not automatically send responses to older messages. Existing review and draft cutoffs remain unchanged. Resume preserves the live cutoff and scan cursor. Automatic sending defaults OFF on upgrade. Draft monitoring alone never enables sending.

Netlify's scheduled function runs every five minutes on the published production deployment, with no laptop required. OAuth authorization must remain valid and hosting must remain available. Incoming messages are evaluated as conversations; already staff-answered threads are skipped.

## Customer replies

- A narrow, explicit pre-purchase question about brand, material, colour, finish or model may receive a direct extractive answer only when the existing strict evidence checks pass. These checks require a single-message conversation, matching seller/item, a complete active single-variant listing, one structured value, and a title/description consisting of consistent structured facts.
- Other eligible questions receive the standard holding response: “Thank you for contacting VirVentures. Our team will review your query and get back to you within 24–48 hours.” Prior identical holding replies are suppressed.
- Size, compatibility, ambiguous specifications, order decisions, crucial claims and other unsupported substantive responses require a human-approved reply. Optional AI rewriting from staff-confirmed facts is available in the reply-writing bot. It never automatically dispatches a generated response and does not modify orders.
- Order matching is incomplete. Positive stored order matches and explicit post-order wording route to Nazar; otherwise the catalogue owner reviews the classification. Ambiguous questions can therefore require manual reassignment outside the app.

## Owner notifications — no time-of-day restrictions

Catalogue/general questions: **anangsha.h@virventures.com**.
Post-order issues: **md.nazar@virventures.com**.
Sender: **team@virventures.com**.

A successful customer send queues a receipt email with the exact reply, customer question, conversation, listing link and status. Holding/suppressed responses also ask the owner to review a substantive response. A successful approved follow-up queues another receipt. Notifications are routed to the responsible owner, not automatically copied to both people. Existing older-message manual sends also queue receipts once the notification workflow is enabled.

Approval emails link to the authenticated dashboard. Replies to emails do NOT approve or send customer messages. Reviewers need authorized access to the existing shared administrator dashboard; per-person identity/role enforcement is not implemented. The responsible reviewer loads current conversation/listing evidence, enters the exact response, attests its accuracy and confirms sending. Approval drafts expire after 15 minutes. Changed conversation/listing evidence blocks dispatch. The confirmed holding message from that same job is the only staff reply excluded when checking whether its substantive follow-up is still eligible.

## Reliability and pause controls

Customer and email reservations commit to encrypted Postgres before external dispatch. Reservations, unknown outcomes and approved attempts are not blindly retried. An uncertain customer send or email delivery pauses automatic replies. Inspect eBay Sent, Gmail Sent and dashboard logs; manual reconciliation remains necessary. A stable email Message-ID helps investigation but is not treated as provider-side idempotency. No exactly-once guarantee is claimed.

After an interrupted customer send remains reserved for ten minutes, a later worker marks it uncertain, queues an alert, and pauses automatic replies. Interrupted email reservations remain visible and block further automatic replies. If Gmail is unavailable, an email cannot report its own failure; the dashboard log is authoritative. Notification emails may run on the next five-minute tick, including after manual sends. Already queued notifications continue when customer replies are paused.

Pause automatic replies prevents new automated dispatches once the setting is observed; an external request already in flight cannot be recalled. Pause scheduled checks stops subsequent scans. There is no atomic compare-and-send operation at eBay, so avoid simultaneous staff replies to the same conversation.

The scanner fetches up to 1,000 summaries for a fixed window, saves that batch, and reviews ten conversations per scheduled invocation. It advances the cursor only after the saved batch is processed. Each new window advances at most one hour with a ten-minute overlap. Failed/incomplete scans pause without advancing. Larger windows exceeding the pagination cap require intervention; sustained traffic can create backlog. Indexing delays beyond the overlap may be missed. This is bounded polling, not a guarantee of exhaustive real-time coverage. Dashboard status displays remaining batch progress and failures.

## Security and configuration

Only pinned Production test 2 eBay credentials are permitted. Production Key Set 1 remains excluded. API mutations require administrator authentication, exact production origin and CSRF protection. Approval tokens are session-bound. Background tasks require job-bound HMAC signatures. All cloud state, credentials, drafts, send ledgers and notifications are encrypted in Postgres. Public dist contains only browser assets. This package includes no credentials, customer data or database.

Preserve REPLYASSIST_ORIGIN, REPLYASSIST_ENCRYPTION_KEY (64 hex characters), and REPLYASSIST_ADMIN_PASSWORD (at least 24 characters). Replacing the encryption key loses access to existing encrypted state. Gmail uses a Web application client and /api/google-callback at the production origin.

## Validation

79 offline tests pass, including sender/recipient restrictions, 24/7 routing, durable reservations, duplicate suppression across snapshot restart, staff-reply checks, approval/session/changed-evidence rejection, catalogue facts, email uncertainty, interrupted-send recovery and frozen batch progress. The existing legacy engine and generic provider sends remain blocked; new cloud dispatch paths are separate and explicit.

The user reported the earlier manual reply build working. This new automatic/customer-email workflow has NOT been deployed or validated live by Codex. After deployment, verify a controlled new conversation, owner receipt, approval path, scheduled execution and pause control before relying on unattended operation.

## Reply-writing bot update — 29 October 2026

After deploying the update, open an unresolved automatic holding reply or a previously sent manual holding reply using **Open reply-writing bot**. This also supports older manual jobs by recovering their saved conversation reference. If the historic record has no listing reference, listing evidence may be absent: verify the exact product from your vendor source.

1. For Claude, add `ANTHROPIC_API_KEY` with your private Claude API key and `REPLYASSIST_AI_PROVIDER=anthropic` in Netlify Production environment variables, Functions scope. Your Claude Console organization needs credits and model access. OpenAI remains supported with `REPLYASSIST_AI_PROVIDER=openai` and `OPENAI_API_KEY`. Do not replace the existing ReplyAssist encryption key, administrator password or OAuth secrets.
2. Claude defaults to `claude-sonnet-5-5`; optionally set `REPLYASSIST_CLAUDE_MODEL` to another accessible structured-output model. OpenAI uses `REPLYASSIST_AI_MODEL` (default `gpt-4o-mini`). No cross-provider fallback occurs if credits run out. Redeploy after changing environment variables. No key is included in this package.
3. Open the relevant conversation in the bot. Enter the exact product/model, a vendor/source reference and date, and the confirmed facts in English or Hindi.
4. Resolve units and product conflicts explicitly. For the Chevy example, 6.5 inches is different from 6.5 feet; the earlier listing described running boards rather than a truck bed. The bot must not silently convert or assume availability.
5. Generate a draft. If clarification is requested, correct the facts and regenerate. Review every factual claim, then approve the exact text to send. The app rechecks conversation/listing changes before dispatch.

The AI receives the staff facts and conversation/listing context, not OAuth tokens or application secrets. Both providers use structured output; OpenAI also uses `store:false`. It has no tools and cannot send messages. Unsupported source excerpts, new numeric values and changed measurement units are rejected. These guards are not a complete semantic proof: a human must still review the wording and product identity. There is no email-instruction ingestion in this version; replying to notification emails does not draft or approve a customer response.

Each successful approved reply queues its owner notification through the existing durable outbox. Delivery health now separates pending, accepted and uncertain emails. **Check uncertain email outcomes** searches Gmail Sent for the existing stable Message-ID without resending; no match does not prove the original was unsent. It checks up to five attempts older than ten minutes. A recovered match does not silently restart paused automatic replies.

Status investigation on 29 October: the connected team@virventures.com mailbox search found only the two ReplyAssist receipts from 28 October (15:45 and 15:50 IST), and no later matches. The live dashboard was inaccessible through the browser connection, so the current cause of missing subsequent notifications is not established. The delivery-health update improves diagnostics; it is not evidence that the live failure is fixed.

Validation: 83 offline tests passed, including mocked AI schema, refusal/incomplete output, session/expiry, unit/number/source guards, legacy manual holding follow-up and read-only email reconciliation. Live model quality and the new UI must be validated after deployment. No live customer messages or emails were sent during implementation.
