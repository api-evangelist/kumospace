---
name: kumospace-manage-subscription
description: Manage a Kumospace space's Stripe-backed subscription — read plan configuration, open checkout and billing-portal sessions, and cancel or renew.
api: kumospace:api
generated: '2026-07-19'
method: generated
source: openapi/kumospace-api-openapi.json
base_url: https://api.kumospace.com
operations:
  - GetSubscriptionConfig
  - GetFreeTrialInfo
  - CreateCheckoutSession
  - CreatePortalSession
  - SetupSubscription
  - SetPaymentSetupCompleted
  - CancelSubscription
  - RenewlSubscription
---

# Manage a Kumospace subscription

Move a space between trial and paid state, and hand billing management back to the customer.

## Before you start

- All operations require `Authorization: Bearer <firebase-id-token>` and act on the caller's space.
- Billing is **Stripe-backed and redirect-based**. `AutomaticStripeSubscription` hangs off `ApiSpace`, and
  the checkout and portal operations return hosted Stripe URLs. You do not handle card data, and there are
  no card fields anywhere in the Kumospace API.
- Subscription state is authoritative on `ApiSpace`: `tier` (`SubscriptionTier`), `subscription`,
  `memberLimit`, `memberMinimum`, `isInternalFreeTrialActive`, `isReverseTrial` and
  `cancelledReverseTrial`. Re-read the space to confirm any change.
- **No idempotency key exists.** Never blindly retry `CreateCheckoutSession`, `CancelSubscription` or
  `RenewlSubscription` — re-read `GetSpaceByName` and branch on actual state instead.

## Steps

1. **Read what is on offer.** `GetSubscriptionConfig` (`GET /v1/payments/config`) returns the available
   plan configuration. Do not hardcode tiers or prices.

2. **Check trial state.** `GetFreeTrialInfo` (`GET /v1/spaces/{name}/trial`) alongside
   `isInternalFreeTrialActive`, `internalFreeTrialScheduledExpirationMs` and `isReverseTrial` on the
   space. A space in an active reverse trial is already in a paid-feature state and needs a different
   path from a cold upgrade.

3. **Start a purchase.** `CreateCheckoutSession` (`POST /v1/payments/checkout`) returns a hosted Stripe
   Checkout URL. Redirect the customer there; do not attempt to complete payment through the API.

4. **Or set up a reverse trial.** `SetupSubscription` (`PUT /v1/payments/reverse-trial/setup`), followed
   by `SetPaymentSetupCompleted` (`PUT /v1/payments/setup`) to mark setup done.

5. **Confirm out of band.** There is **no webhook and no callback** from Kumospace telling you checkout
   completed. After the redirect returns, poll `GetSpaceByName` (`GET /v1/spaces/{name}`) and read `tier`
   and `subscription` until the change lands. Do not treat the redirect itself as proof of payment.

6. **Hand over self-service.** `CreatePortalSession` (`POST /v1/payments/billingPortal`) returns a Stripe
   Billing Portal URL where the customer manages payment methods, invoices and cancellation themselves.
   Prefer this over building your own billing UI.

7. **Cancel or renew.**
   - `CancelSubscription` (`POST /v1/payments/subscription/cancel`)
   - `RenewlSubscription` (`POST /v1/payments/subscription/renew`) — note the typo in the published
     `operationId`; it is `Renewl`, not `Renewal`, and generated clients will carry the misspelling.

## Notes and gotchas

- `memberLimit` and `memberMinimum` are tier-bound. A downgrade with more members than the target tier
  allows has no declared error response, so verify member count with `GetUserRoles` before cancelling or
  changing plans.
- HIPAA is a paid-plan feature, surfaced as `hipaaEnabled` on `ApiSpace`. Cancelling a paid plan has
  compliance implications for a space that relies on it.
- `tierNameOverride` exists on `ApiSpace`, so the displayed plan name may not match the underlying `tier`.
  Branch logic on `tier`, never on the display string.
