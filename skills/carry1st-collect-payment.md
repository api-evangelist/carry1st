---
name: Collect a payment with Pay1st
description: Authenticate, discover local payment methods, create a signed payment request, and confirm the payment status through the Pay1st Gateway.
api: openapi/carry1st-pay1st-gateway-openapi.yml
operations: [generateAccessToken, listPaymentMethods, createPaymentRequest, queryPaymentStatus]
---

# Collect a payment with Pay1st

Use this skill to take a one-off payment from an African customer through the Pay1st Gateway,
where Carry1st is Merchant of Record.

## Prerequisites
- An `API Key`, `API Secret`, and `Signing Key` generated in the Pay1st Console.
- Base URL: `https://api-gateway.carry1st.com` (production) or
  `https://api-gateway.platform.stage.carry1st.com` (sandbox).

## Steps

1. **Authenticate — `generateAccessToken`**
   `POST /api/pay1st/auth/token` with header
   `Authorization: Basic base64(API_KEY:API_SECRET)`,
   `Content-Type: application/vnd.carry1st.payments.partnerauthentication+json`,
   body `{"role":"API_USER"}`. Store the returned `accessToken` (and `refreshToken`).

2. **(Optional) discover methods — `listPaymentMethods`**
   `GET /api/pay1st/payments/methods?countryCode=NG` with
   `AccessToken: Bearer <accessToken>`. Use the returned `channelId`/`channelName` list to
   present local options.

3. **Sign and create the payment — `createPaymentRequest`**
   Build the request JSON (amount, ISO-2 `countryCode`, ISO-4217 `currencyCode`, a
   `partnerReference` that is **unique for the lifetime of the integration**, `products[]`,
   `metadata[]`). Generate `X-SIGNATURE` = lowercase hex HMAC-SHA256 of
   `<X-TIMESTAMP><requestJSON>` keyed by the Signing Key. Send
   `POST /api/pay1st/payments/create` with `AccessToken`, `Content-Type: application/vnd.carry1st.payments.payment+json`,
   `X-SIGNATURE`, and `X-TIMESTAMP` (ISO-8601, within 5 minutes). A `201` returns the Pay1st
   `reference` with `status: NEW`. Redirect the customer to the hosted-payment URL / callback flow.

4. **Confirm — `queryPaymentStatus`**
   `GET /api/pay1st/payments?reference=<reference>` with `AccessToken`. Poll (or rely on the
   Payment Webhook Events / callback URLs) until `status` reaches a terminal value:
   `SUCCESSFUL`, `FAILED`, `CANCELLED`, or `ABANDONED`. Fulfil the order only on `SUCCESSFUL`.

## Rules
- `partnerReference` is the idempotency/reconciliation key — never reuse it.
- Every mutating request must carry a fresh valid `X-SIGNATURE` + `X-TIMESTAMP`; a stale
  timestamp (>5 min) or mismatched signature returns `400`.
- Errors return `{errorMessage, errorCode, sessionId}`; send `sessionId` to the Pay1st
  Implementation Manager to troubleshoot. See `errors/carry1st-problem-types.yml`.
- Back off on `429`.
