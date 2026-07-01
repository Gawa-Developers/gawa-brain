# 2026-07-01 — PayMongo Business Verification vs Zero-Registration Rail (travel-it-all)

## Summary

Worked through PayMongo sandbox setup for a travel booking site (fixed real webhook signature-verification and intentId bugs along the way, verified a live QRPh sandbox payment end-to-end). Discussion then turned to going live: the business isn't registered yet, most transactions are Cash on Delivery, and online payments are expected to stay occasional for now — so the question was whether real online payments could be received before formalizing the business.

## Facts Confirmed (PayMongo, as of 2026-07)

- Live API keys are gated by account tier: unverified ("M1/Basic") accounts cannot access live keys or accept live payments at all, regardless of payment method (QRPh included).
- KYC (identity) + KYB (business docs) verification is mandatory for live mode and cannot be skipped, typically ~14 business days after submission.
- Activation (can accept payments) and Enabled Wallet (can pay out to a bank/e-wallet) are two *separate* reviews — full activation doesn't automatically unlock withdrawal.
- Card and e-wallet (GCash/Maya/GrabPay) payment methods require a separate "activate payment method" step, account-owner-gated, on top of the above.
- QRPh is enabled by default even in test/sandbox mode with no verification — this is what powered the successful end-to-end sandbox test (real Payment Link → real sandbox payment → webhook → booking confirmed).

## Facts Confirmed (PayPal Personal, Philippines, as of 2026-07)

- A Personal PayPal account can receive money with no business registration and no verification required upfront.
- Withdrawal only requires linking a PH bank account/card (personal identity level, not business KYB) — standard transfer 1–5 business days, ₱50 fee under ₱7,000.
- Caveat: PayPal's terms frame Personal accounts for "occasional" use; sustained business-pattern activity (recurring payments from many distinct senders) risks a retroactive hold pending verification — the same underlying requirement PayMongo asks for upfront, just enforced reactively instead.

## Decision

Promoted to [decisions.md — D7](../../../memory/decisions.md#d7-prove-payment-demand-on-a-zero-registration-rail-before-committing-to-a-verified-processor): use PayPal Personal for the occasional online payment now; keep the PayMongo integration sandbox-tested and code-complete; start PayMongo business verification only once online volume grows enough that it's clearly no longer "occasional" — treat that volume signal as the trigger, not a calendar date.

## Context Specific to This Case

travel-it-all is COD-first — online payment is a secondary channel whose real demand is still unproven. The project owner explicitly wants to avoid sinking registration effort into a channel that might not pay off, while keeping the technical integration ready so flipping to live PayMongo later is a config change, not a build.
