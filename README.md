# ReplyAssist Cloud Review

This is a Netlify adaptation of the uploaded review build, not a live response service.
Customer replies, Gmail sends, continuous monitoring and activation remain blocked at the provider and application boundaries. The grounded AI pipeline and complete order matching are still absent.

## Deployment

The repository transport uses `source.json`, a plain-text map of relative filenames to source contents, plus `unpack.mjs`. The Netlify build expands that map, runs the offline tests, copies only `cloud-public` to `dist`, and bundles the backend functions. This keeps source, tests, Windows launchers, and credentials out of the public web directory. For normal git development, run `node unpack.mjs` and work with the expanded files; rebuild the source map before committing through this transport.

Netlify settings: build `node unpack.mjs && npm run build`, publish `dist`, Node 24. The @netlify/database dependency requests managed Postgres; migration files are expanded before build finishes. Confirm database provisioning and migration success in Netlify before using the app. Existing plan credits/usage apply.

Set production Function environment variables in Netlify:
- REPLYASSIST_ORIGIN: the exact https origin of the production site, with no path.
- REPLYASSIST_ENCRYPTION_KEY: 32 cryptographically random bytes encoded as 64 hexadecimal characters. Preserve this key securely; replacing it makes existing encrypted state unreadable.
- REPLYASSIST_ADMIN_PASSWORD: a unique password at least 24 characters long. This is the private dashboard password. Never commit it.

The app fails closed until configured. Redeploy after changing Function environment values. Preview origins cannot authenticate or access the production dashboard.

## Architecture and limits

API routes require a signed Secure/HttpOnly cookie. POST routes require same-origin JSON and a session-bound CSRF token. Requests are rate-limited at Netlify. All cloud state, including OAuth tokens and conversation evidence, is AES-256-GCM encrypted before persistence. Postgres serializes mutations with a row lock. A transient in-memory SQLite store preserves the original review logic; its state is saved inside the encrypted Postgres record before successful mutation responses.

Long manual checks enqueue authenticated background functions. Internal triggers require a job-bound HMAC. Replayed finished tasks are ignored, and concurrent runs serialize through Postgres. Provider reads have an overall three-minute budget. Status reads use a read-only transaction. This single-record review store is deliberately for bounded validation and is not a scalable production inbox ledger.

Google requires a Web application OAuth client with `https://YOUR-SITE/api/google-callback`, authorized as team@virventures.com. Windows Desktop client JSON and DPAPI files cannot be used here. eBay accepts only Production test 2, with the existing test 2 redirect name; Production Key Set 1 is excluded. Complete a fresh authorization on this cloud instance. No old code or local vault is uploaded.

Sequence: configure host secrets, sign in, authorize providers, verify current seller/listing and Gmail, establish immutable cutoff, check message access, run bounded dry run. Only the first 24 hours after that cutoff are eligible; at most 10 conversations per run. This is a validation sample, not ongoing coverage. Preserve the original Windows ledger separately. Unknown historical sends remain unresolved, and all proposals require recovery review.

## Validation status

45 offline tests pass, including eight cloud-specific tests. These test session expiry/tampering/password rotation, CSRF/origin checks, encryption integrity, state roundtrip/proposal deduplication, route locks, credential restrictions, body limits and fail-closed configuration. No live Postgres transaction, Netlify deployment, browser session, OAuth or provider read has yet been validated. Do not claim automatic operation until those connected checks are complete.
