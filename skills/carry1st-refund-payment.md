---
name: Refund a Pay1st transaction
description: Authenticate and issue a full or partial refund against a processed Pay1st transaction, then track it to a terminal status.
api: openapi/carry1st-pay1st-gateway-openapi.yml
operations: [generateAccessToken, createRefundRequest]
---

# Refund a Pay1st transaction

Use this skill to refund a payment that Pay1st has already processed. The Refund API is
documented as **beta**.

## Steps

1. **Authenticate — `generateAccessToken`**
   `POST /api/pay1st/auth/token` (HTTP Basic with API Key/Secret, body `{"role":"API_USER"}`)
   and keep the `accessToken`.

2. **Sign and request the refund — `createRefundRequest`**
   `POST /api/pay1st/transactions/refunds/create` with headers `AccessToken: Bearer <accessToken>`,
   `Content-Type: application/vnd.carry1st.payments.refund+json`, `X-SIGNATURE`, `X-TIMESTAMP`.
   Body requires `transactionReference` (the `C-<digits>-T` id from the original payment);
   optionally `amount` (omit for a full refund), `partnerRefundReference` (your unique refund id),
   `reason`, and `refundWebhookUrl`. A `201` returns `refundReference` with `status: PENDING` and
   a `handlingType` of `PSP_API`, `PSP_MANUAL`, or `NO_REFUND`.

3. **Track to terminal status**
   The refund progresses via the Refund Webhook through `PROCESSING` to a terminal
   `SUCCESSFUL`, `FAILED`, or `REJECTED`. Reconcile against your `partnerRefundReference`.

## Rules
- Sign exactly as for payments: lowercase hex HMAC-SHA256 over `<X-TIMESTAMP><requestJSON>`
  keyed by the Signing Key; timestamp within 5 minutes.
- If no `amount` is supplied, Pay1st processes a **full** refund.
- Handle `{errorMessage, errorCode, sessionId}` errors; see `errors/carry1st-problem-types.yml`.
