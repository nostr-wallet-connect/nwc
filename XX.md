NWC-XX
======

Universal Payment Lookup
------------------------

`draft` `optional`

## Summary

This specification defines the optional `lookup_payment` method for looking up incoming and outgoing payments across multiple Bitcoin payment protocols.

It defines records for:

- BOLT11 payments
- BOLT12 payments
- regular on-chain payments
- BIP-352 silent payments

The record shape can be extended by future optional specifications with additional `payment_type` values and type-specific fields.

## Motivation

Applications need to reconcile a payment after a timeout, reconnect, retry, or notification without knowing how the wallet executed or received it.

The `lookup_invoice` method in NWC core is specific to Lightning invoices. A general lookup method also needs stable wallet identifiers, unambiguous on-chain output identifiers, BOLT12 fields, and silent-payment fields.

This method is based on the `lookup_payment` command in [NIP-101](https://github.com/matbalez/universal-lightning-wallet/blob/main/NIP-101.md), adapted to existing NWC field names and extended to non-Lightning payments.

## Dependencies

Implementations use the relevant specification for each payment type:

- [BOLT11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)
- [BOLT12](https://github.com/lightning/bolts/blob/master/12-offer-encoding.md)
- [BIP-321](https://github.com/bitcoin/bips/blob/master/bip-0321.mediawiki) where a payment was initiated from a BIP-321 URI
- [BIP-352](https://github.com/bitcoin/bips/blob/master/bip-0352.mediawiki) for silent payments

The `metadata` field follows [06.md](06.md).

## Discovery

A wallet service supporting this specification MUST advertise `lookup_payment` as a supported method in its NWC info event and `get_info` response.

Once this specification has a final extension number, the wallet service SHOULD also advertise that number according to NWC core extension discovery.

## Method

### `lookup_payment`

Looks up one payment record visible to the requesting connection.

Request using the preferred wallet-scoped identifier:

```yaml
{
    "method": "lookup_payment",
    "params": {
        "transaction_id": "wallet-scoped-id"
    }
}
```

The request MUST use exactly one of these selector forms:

| Selector form | Payment types | Requirements |
| --- | --- | --- |
| `transaction_id` | all | Exact wallet-scoped identifier returned by another NWC method. |
| `payment_hash` | `bolt11`, `bolt12` | 32-byte payment hash encoded as lowercase hex. |
| `invoice` | `bolt11`, `bolt12` | Encoded BOLT11 or BOLT12 invoice. |
| `offer_id`, optionally with `payment_hash` | `bolt12` | 32-byte BOLT12 offer ID encoded as lowercase hex. `payment_hash` SHOULD be included for a reusable offer. |
| `txid`, optionally with `vout` | `onchain`, `silent_payment` | Transaction ID encoded as lowercase hex. `vout` is the zero-based output index and SHOULD be included for a transaction containing more than one wallet-visible payment. |

Examples of alternative selectors:

```yaml
{ "payment_hash": "31afdf1c..." }
{ "invoice": "lnbc50n1..." }
{ "offer_id": "0123456789abcdef...", "payment_hash": "31afdf1c..." }
{ "txid": "89abcdef...", "vout": 1 }
```

`vout` MUST NOT be present without `txid`. When `payment_hash` accompanies `offer_id`, it qualifies the offer lookup and does not constitute a second selector form.

The wallet service MUST match selectors only against records visible to the requesting connection. It MUST NOT reveal whether an inaccessible record exists.

Every successful lookup MUST resolve to exactly one payment record. A wallet service MUST NOT select an arbitrary payment when an offer, transaction, or other selector matches multiple records.

Response:

```yaml
{
    "result_type": "lookup_payment",
    "result": {
        "transaction_id": "wallet-scoped-id", // stable identifier for this payment record
        "type": "outgoing", // "incoming" or "outgoing"
        "state": "settled", // "pending", "accepted", "settled", "failed", "expired", or "canceled"
        "payment_type": "bolt12", // "bolt11", "bolt12", "onchain", or "silent_payment"
        "amount": 123000, // payment amount in msats
        "fees_paid": 1000, // fees attributed to this payment in msats, optional
        "bip321": "bitcoin:?lno=lno1...", // associated BIP-321 URI, optional

        "invoice": "lni1...", // encoded BOLT11 or BOLT12 invoice, optional
        "offer": "lno1...", // encoded BOLT12 offer, optional
        "offer_id": "0123456789abcdef...", // BOLT12 offer ID, optional
        "payment_hash": "31afdf1c...", // Lightning payment hash, optional
        "preimage": "0123456789abcdef...", // Lightning payment preimage, optional
        "payer_proof": "lnp1...", // BOLT12 payer proof, optional
        "payer_note": "string", // BOLT12 payer note, optional

        "address": "bc1q...", // recipient address for a regular on-chain payment, optional
        "silent_payment_address": "sp1q...", // BIP-352 address used by the sender, optional
        "output_address": "bc1p...", // derived output address for a silent payment, optional
        "script_pubkey": "5120...", // payment output script encoded as lowercase hex, optional
        "txid": "89abcdef...", // on-chain transaction ID, optional
        "vout": 1, // zero-based payment output index, optional
        "confirmations": 6, // confirmations at lookup time, optional
        "required_confirmations": 3, // wallet threshold for the "settled" state, optional
        "block_height": 900000, // confirming block height, optional
        "block_hash": "abcdef...", // confirming block hash, optional
        "confirmed_at": unixtimestamp, // first confirmation time, optional

        "description": "string", // payment description, optional
        "description_hash": "string", // payment description hash, optional
        "created_at": unixtimestamp, // wallet record creation time
        "expires_at": unixtimestamp, // payment request expiration time, optional
        "settled_at": unixtimestamp, // time the wallet moved the record to "settled", optional
        "failure_reason": "string", // optional unless state is "failed"
        "metadata": {} // optional metadata as defined in 06.md
    }
}
```

All fields not explicitly marked optional are REQUIRED.

`transaction_id` MUST be stable within the wallet service and MUST identify the same payment in responses from other NWC methods. It identifies a payment record, not necessarily an entire on-chain transaction. A batched on-chain transaction can therefore have multiple payment records with different `transaction_id` and `vout` values but the same `txid`.

`amount` and `fees_paid` use millisatoshis for every payment type. On-chain satoshi amounts MUST be converted exactly by multiplying by 1000.

`fees_paid` is the fee attributed by the wallet to this payment. It MAY be omitted when the fee is unavailable or cannot be meaningfully allocated, such as for one payment in a batched on-chain transaction.

`bip321` MAY contain the associated BIP-321 URI when the payment was initiated from or represented by one.

## Payment states

- `pending`: The payment or receive target exists but has not reached the wallet's settlement policy.
- `accepted`: A hold invoice has been accepted but not settled.
- `settled`: The wallet considers the payment final under its policy.
- `failed`: An outgoing payment attempt ended unsuccessfully.
- `expired`: A receive target expired without settlement.
- `canceled`: The payment or receive target was canceled.

For on-chain payments, `confirmations` reports current chain depth. `required_confirmations` reports the wallet's current threshold for moving the record to `settled`. A wallet using a fixed confirmation threshold SHOULD return `required_confirmations`. A chain reorganization MAY move an on-chain record from `settled` back to `pending`.

## Payment-type fields

### BOLT11

A `bolt11` record SHOULD include `invoice` and MUST include `payment_hash`. It SHOULD include `preimage` when settled and available. Invoice descriptions and expiry use the common `description`, `description_hash`, and `expires_at` fields.

### BOLT12

A `bolt12` record MUST include `payment_hash`. It SHOULD include the paid BOLT12 `invoice` and `offer_id` when available. It MAY include `offer`, `preimage`, `payer_proof`, and `payer_note`.

Because a BOLT12 offer can be reusable, `offer_id` alone is not guaranteed to identify one payment. Clients SHOULD retain the returned `transaction_id` or combine `offer_id` with `payment_hash` for later lookup.

### Regular on-chain

An `onchain` record MUST include `txid` once the transaction has been broadcast or detected and SHOULD include `vout` and `address` for the payment output. It MAY include `script_pubkey` when the output has no standard address or the exact output script is useful. It SHOULD include confirmation fields once confirmed.

An outgoing record MAY be returned before broadcast. In that case `txid`, `vout`, and confirmation fields can be absent while `state` is `pending`.

### Silent payment

A `silent_payment` record follows the on-chain requirements and MUST be identified as a separate payment type so clients do not treat its reusable BIP-352 address like an ordinary output address.

It SHOULD include `silent_payment_address` when the wallet retained or can reconstruct the address used by the sender. Once detected or broadcast, it MUST include `txid` and `vout` and SHOULD include the derived `output_address`.

Clients MUST NOT use `silent_payment_address` as a unique payment identifier. BIP-352 addresses can be reused and the actual payment output is derived per transaction.

## Errors

- `BAD_REQUEST`: The selector form or an identifier is invalid.
- `NOT_FOUND`: No payment visible to this connection matches the selector.
- `MULTIPLE_MATCHES`: More than one payment visible to this connection matches the selector. The client should retry with `transaction_id`, or with `payment_hash` or `vout` as a qualifier where applicable.

The wallet service can also return applicable NWC core errors.

## Relationship to other specs

- NWC core `lookup_invoice` remains available for BOLT11-specific compatibility.
- [05.md](05.md) defines `list_transactions`; implementations supporting both methods SHOULD use the same transaction record identifiers and field semantics.
- [321.md](321.md) defines BIP-321 `pay` and `receive`; its `transaction_id` refers to the same wallet-scoped identifier used here, and its `instruction_type` maps to `payment_type` for `bolt11` and `bolt12` records.
- [02.md](02.md) notifications are state-change hints. Clients can use `lookup_payment` to reconcile the authoritative current record after receiving or missing a notification.
