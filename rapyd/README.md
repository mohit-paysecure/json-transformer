# Rapyd Integration

## Files
- `rapyd_payin.json` — Paysecure transformer config for `POST /v1/payments` (card payment, auth+capture)
- `rapyd_postman_collection.json` — Postman collection for raw API testing against Rapyd sandbox

## Rapyd request signing (recap)

Every request to Rapyd needs four headers:
- `access_key`
- `salt` (random 8-16 chars)
- `timestamp` (Unix seconds, must be within 60s of server time)
- `signature` = `BASE64( HEX( HMAC-SHA256( http_method_lowercase + url_path + salt + timestamp + access_key + secret_key + body_string ) ) )`

`body_string` is the **compact JSON** body sent in the request (no whitespace), or empty string for GET — NOT `{}`.

## Using the Postman collection

1. Import `rapyd_postman_collection.json` into Postman
2. Open collection variables → set:
   - `base_url` = `https://sandboxapi.rapyd.net` (or `https://api.rapyd.net` for production)
   - `access_key` = your Rapyd access key
   - `secret_key` = your Rapyd secret key
3. Run "Sanity — List Countries" first to verify your signing works (should return 200)
4. Then run "Create Payment — Card", which saves `last_payment_id` for chaining
5. Subsequent requests (Retrieve, Capture, Refund) use the saved id

The collection-level pre-request script auto-signs every request. Don't add `access_key`/`salt`/`timestamp`/`signature` headers manually — the script will set them.

## Paysecure DB setup for Rapyd

### `payment_bank` row

| Column | Value |
|---|---|
| `name` | `rapyd` |
| `live_url` | `https://sandboxapi.rapyd.net` (TEST) or `https://api.rapyd.net` (PROD) |
| `is_active` | `1` |

### `payment_bank_mid` row

| Column | Value |
|---|---|
| `mid` | `rapyd` |
| `allowed_card` | `VISA,MASTER,AMEX,DISCOVER,JCB` |
| `mid_auth_key` | `<access_key>##<secret_key>` — `##`-split for signing ops in JSON |
| `is_active` | `1` |

### `transformation_config` row

| Column | Value |
|---|---|
| `payment_bank_id` | `<rapyd_bank_id>` |
| `payment_flow` | `PAYIN` |
| `config_json` | contents of `rapyd_payin.json` |

## Known caveat: body byte-match for signing

The `rapyd_payin.json` builds the body TWICE:
1. Once as the visible body (sent to Rapyd as HTTP body)
2. Once inside a hidden `_rapydBodyString` node that stringifies the same structure

Both must produce **byte-identical JSON** for the signature to verify. Things that can break this:
- Different key insertion order between the two copies (both use `KEY_VALUE_LIST_TO_OBJECT` — should be deterministic, but verify)
- Different value transformations (e.g., one path adds STRING_TO_BOOLEAN, the other doesn't)
- The framework's final `JSONObject.toString()` on the body vs. `STRINGIFY_JSON`'s output — these should match if both use compact JSON formatting

If signature verification fails (Rapyd returns `UNAUTHORIZED_API_CALL`), enable a debug log to print both the `_rapydBodyString` and the actual HTTP body sent, diff them, then adjust the mappings until they match.

## payment_method.type mapping

Rapyd uses country+brand specific identifiers like `us_visa_card`, `gb_mastercard_card`, `mx_visa_card`. The current JSON builds this as:
```
lowercase(purchase.client.country) + "_" + lowercase(cardInfo.cardBrand) + "_card"
```

Verify the cardBrand values (`VISA`, `MASTER`, `AMEX`) lowercase correctly to `visa`, `master`, `amex` — and that Rapyd accepts `master` (Rapyd's docs sometimes use `mastercard`). If you see "INVALID_PAYMENT_METHOD_TYPE" errors, hardcode a `VALUE_MAPPER` from brand to Rapyd's exact key.

Use the Postman "List Payment Method Types — by country" request to enumerate valid types for each country.

## 3DS flow

If Rapyd's response includes `data.redirect_url`, the customer must complete 3DS there. After completion they return to:
- `complete_payment_url` (success) — defined in PAYIN body
- `error_payment_url` (failure) — defined in PAYIN body

Both currently point to `paysecure /custRedirect/{bankId}/{purchaseId}/?status=...`. Paysecure's redirect handler at that path should then poll Rapyd's `GET /v1/payments/{id}` (via INQUIRY config, when written) to resolve final status.

## Files to add later

- `rapyd_inquiry.json` — `GET /v1/payments/{id}` for status polling
- `rapyd_refund.json` — `POST /v1/refunds`
- `rapyd_webhook.json` — signature verification + status mapping for async events
