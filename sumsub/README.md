# Sumsub — KYC provider (sandbox)

Non-seamless IDV. We create an applicant keyed on **our customerId**, mint a short-lived WebSDK access
token, serve the SDK from our own origin, and take the verdict from `KYC-INQUIRY` (webhook only wakes
the record).

## Config rows

| file | `payment_flow` |
| --- | --- |
| `kyc_initiate.json` | `KYC-INITIATE` (4 steps) |
| `kyc_inquiry.json` | `KYC-INQUIRY` |
| `kyc_webhook.json` | `KYC-WEBHOOK` |

All three parse cleanly into `List<ConfigJson>` — verified with the same ObjectMapper
`KycProviderResolver` uses, so every operation name resolves.

## Auth key layout

`payment_bank_mid.mid_auth_key`, `##`-separated — the same convention every PSP here uses. Configs
`SPLIT_STRING` then `GET_ELEMENT_AT_INDEX`:

| index | value |
| --- | --- |
| 0 | App Token (`sbx:...`) |
| 1 | Secret Key — signs S2S requests |
| 2 | Webhook secret — verifies `x-payload-digest` |
| 3 | Verification level name (e.g. `basic-kyc-level`) |

**Indexes 2 and 3 do not exist yet** — the webhook secret and the level must be created in Cockpit
first. Until then `KYC-WEBHOOK` cannot verify a signature and `KYC-INITIATE` cannot name a level.

## Signing

Every S2S call sends `X-App-Token`, `X-App-Access-Ts`, `X-App-Access-Sig`, where the signature is
`HMAC-SHA256(ts + METHOD + pathWithQuery + body)` hex, keyed on index 1.

No Java was needed: `{{kycSign.pathWithQuery}}` and `{{kycSign.body}}` come from `KycSignMaterial`, and
`KycFlowImpl` deliberately resolves the URL *before* headers so a config can sign the resolved path.

## Flow

**`KYC-INITIATE`** — 4 steps:

0. `GET /resources/applicants/-;externalUserId={customerId}/one` — reuse an existing applicant.
   `skipPspCount` is `1` on HTTP 200 (jump the create step), `0` otherwise. This is what makes a
   resubmission reuse the same applicant instead of 409-ing.
1. `POST /resources/applicants?levelName=…` — create, `externalUserId = our customerId`.
2. `POST /resources/accessTokens?userId=…&levelName=…&ttlInSecs=1800` — WebSDK token.
3. compute-only (`skipPspCall`) — `GENERATE_HTML` in `SUMSUB_WEBSDK` mode emits `redirectHtml`,
   `status=AWAITING_USER`, `terminal=true`.

The subject reaches it via `GET /kyc/redirect/{kycId}` on our origin, so the access token never goes to
the merchant. Top-level page, not an iframe — the SDK needs the camera.

**`KYC-INQUIRY`** — `GET /resources/applicants/{applicantId}/status`, the only source of a verdict.

**`KYC-WEBHOOK`** — verifies `x-payload-digest` against the raw bytes, emits `providerReferenceId = applicantId`.
Carries no verdict by design; it wakes the record and `KYC-INQUIRY` decides.

## State mapping

`reviewAnswer` + `reviewRejectType` are joined into one key so a single `VALUE_MAPPER` decides:

| Sumsub | our `decision` | our `status` |
| --- | --- | --- |
| `GREEN` | `APPROVED` | `COMPLETED` |
| `RED` + `FINAL` | `DECLINED` | `COMPLETED` |
| `RED` + `RETRY` | `RESUBMISSION_REQUIRED` | `COMPLETED` |
| `reviewStatus: onHold` | `PENDING` | `MANUAL_REVIEW` |
| `reviewStatus: init` | `PENDING` | `AWAITING_USER` |
| `reviewStatus: pending`/`queued` | `PENDING` | `AWAITING_PROVIDER` |

`RESUBMISSION_REQUIRED` is new. It settles the attempt but `resolveSettled` lets a fresh verification
through, so a blurry document does not lock the customer out. `DECLINED` still blocks re-verify.

Your spec's `INITIATED` / `PENDING` / `NOT_FOUND` are not record states here — this module keeps status
and decision on separate axes, so they map to pairs above. `NOT_FOUND` is a 404 from our own API, not a
record state.

## SQL

Values deliberately left as placeholders — do not paste live secrets into this file.

```sql
INSERT INTO payment_bank (name, class_name, is_active, test_url, live_url)
VALUES ('sumsub', 'org.pgs.service.payment.commongateway.Main', 1,
        'https://api.sumsub.com', 'https://api.sumsub.com');

INSERT INTO payment_bank_mid (pp_id, mid, mid_desc, mid_auth_key)
VALUES (<payment_bank.id>, 'sumsub-sbx', 'Sumsub sandbox',
        '<appToken>##<secretKey>##<webhookSecret>##<levelName>');

-- one row per file
INSERT INTO transformation_config (payment_bank_id, payment_flow, config_json)
VALUES (<payment_bank.id>, 'KYC-INITIATE', '<contents of kyc_initiate.json>');
```

`class_name` is `commongateway.Main` only to satisfy the payments resolver; KYC never instantiates it —
`KycProviderResolver` reads the config rows directly.

## Config gotcha that cost a live failure

`COMPUTE_HMAC_SHA failed to convert input to java.util.Map` — the operation takes a `Map`, but a
non-leaf node yields a key-value **list**. Every node whose children carry `key`s must terminate in
`KEY_VALUE_LIST_TO_OBJECT`, including `headers`, `body`, and the HMAC node itself. This is universal
across every PSP config in this repo — check `rapyd/txn_inquiry_payin.json` for the canonical shape.

## Open — needed before this can run

1. **Webhook secret + level name** created in Cockpit → auth-key indexes 2 and 3.
2. **Webhook URL** registered in Cockpit. Either `/kyc/webhook/sumsub`, or `/webhook/sumsub` with a
   `WEBHOOK_ROUTING` row if Sumsub events share the payments URL.
3. **Field names unverified against live responses** — `reviewResult.reviewAnswer`,
   `reviewResult.reviewRejectType`, `reviewStatus`, `applicantId`, `token`, `id`, and the
   `x-payload-digest` header. Written from the API docs; confirm against a real sandbox response and
   adjust the `{{pspResponseN.*}}` paths.
4. **SDK event names** in `SUMSUB_WEBSDK` — `idCheck.onApplicantSubmitted` /
   `idCheck.onApplicantResubmitted`. Confirm against current WebSDK docs.
5. **Token refresh is a hard stop.** The refresh callback rejects, because minting a token needs our
   signed S2S call. A subject who idles past `ttlInSecs=1800` must re-enter through `/kyc/verify`.
6. Open questions from the checklist: webhook retry schedule, egress IPs, default token TTL, per-account
   rate limits.
