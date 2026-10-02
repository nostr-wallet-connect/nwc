NWC-XX
======

Commission Payments
-------------------

`draft` `optional`

## Summary

This specification defines commission payments for Nostr Wallet Connect.

A client, typically a marketplace platform, can ask a wallet service to create a **sale invoice**. Once a payer pays it, the wallet service pays a commission to the client, within a **commission policy** set by the user. Both payments share one payment hash, so the client can only collect the commission by revealing the preimage that lets the wallet service collect the sale.

It includes:

- the commission policy
- the `make_commission_invoice` method
- the wallet service's validation, timing and settlement rules
- client behavior for capturing and canceling sales

## Motivation

Marketplaces earn a commission on sales between payers and their users. Today a client can only collect it custodially, through a broad `pay_invoice` permission on the user's wallet, by trusting the user to pay later, or by making the payer pay twice.

This specification adds a narrow, conditional spending permission instead: pay at most a given share of a payment being received, only atomically with that payment. The wallet service enforces it, and the payer pays one ordinary BOLT11 invoice. The same permission covers any share of a received payment owed to a third party, such as affiliate fees, referral splits or service fees.

## Dependencies

Implementations use:

- [02.md](02.md) for notifications
- [03.md](03.md) for the client's commission invoice, a hold invoice on the client's own wallet
- [09.md](09.md) for payment states
- [08.md](08.md) for client-initiated connection creation, optionally

## Discovery

A wallet service supporting this specification MUST:

- advertise `make_commission_invoice` as a supported method in its NWC info event and `get_info` response
- advertise `XX` according to NWC core extension discovery
- include the connection's effective commission policy, with defaults filled in, as `commission_policy` in the `get_info` response

## Commission policy

A commission policy belongs to one connection. It is set by the user and enforced by the wallet service. The client cannot change it.

```yaml
{
    "max_rate_ppm": 100000, // maximum commission rate in parts per million of the sale amount, e.g. 100000 = 10%
    "max_fixed_msat": 0, // maximum fixed amount added to the rate, optional; default 0
    "payees": ["02ab..."], // allowed payee node pubkeys, optional
    "max_routing_fee": { "base_msat": 1000, "rate_ppm": 5000 }, // optional; default set by the wallet service
    "budget_msat": 50000000, // maximum commissions plus routing fees per renewal_period, optional
    "renewal_period": "monthly" // "never", "daily", "weekly", "monthly" or "yearly", optional; default "never"
}
```

`max_rate_ppm` is REQUIRED.

For a commission amount `c`, the maximum routing fee is `base_msat + floor(c * rate_ppm / 1000000)`. Routing fees are paid by the user.

If `payees` is set, commission invoices MUST be payable to one of these nodes.

The user MAY change the policy at any time. A change applies only to sale invoices created afterwards. Creating a commission connection or making its policy less strict MUST require explicit confirmation by the user, after showing every policy field in plain language.

The connection's expiry, for example the 08 `expires_at`, also ends the commission permission. On revocation or expiry, the wallet service MUST reject new requests and cancel sale invoices that have not been paid yet. Sales already `accepted` are completed as defined below.

## Connection creation

How the connection is created is outside the scope of this specification. A commission connection SHOULD only permit `make_commission_invoice`, `lookup_invoice`, `get_info` and `payment_received` notifications. It MUST NOT permit any method that sends payments or settles invoices, nor `create_connection`. Clients SHOULD reject connections that do.

A client using [08.md](08.md) MUST list `make_commission_invoice` in `request_methods`, so that a wallet service not supporting this specification declines the request. It adds a `commission_policy` parameter with the URL-encoded, JSON-serialized policy, in either flow, and SHOULD NOT include `max_amount`:

`nostr+walletauth://b889ff5b1513b641e2a139f661a661364979c5beee91842f8f0ef42ab558e9d4?relay=wss%3A%2F%2Frelay.example.com&state=9f2c4a...&request_methods=make_commission_invoice%20lookup_invoice%20get_info&notification_types=payment_received&commission_policy=%7B%22max_rate_ppm%22%3A100000%7D`

The wallet service MUST create the connection with the requested policy or a stricter one chosen by the user, or decline. Unlike `max_amount`, a stricter policy is allowed. The client learns the effective policy from `get_info`.

## Methods

### `make_commission_invoice`

Creates a sale invoice bound to a commission invoice.

Request:
```yaml
{
    "method": "make_commission_invoice",
    "params": {
        "amount": 21000000, // sale amount in msats
        "payment_hash": "string", // generated by the client, shared by both invoices
        "commission_invoice": "string", // client's hold invoice with the same payment hash
        "expiry": 600, // expiry in seconds from time invoice is created for a payment to be initiated
        "description": "string", // invoice's description, optional
        "description_hash": "string", // invoice's description hash, optional
        "metadata": {} // generic metadata, optional
    }
}
```

Response:
```yaml
{
    "result_type": "make_commission_invoice",
    "result": {
        "type": "incoming",
        "state": "pending", // see sale invoice states
        "invoice": "string", // encoded sale invoice
        "description": "string", // invoice's description, optional
        "description_hash": "string", // invoice's description hash, optional
        "preimage": "string", // payment's preimage, present when settled
        "payment_hash": "string", // Payment hash for the payment
        "amount": 21000000, // value in msats
        "fees_paid": 0, // value in msats, always 0
        "created_at": unixtimestamp, // invoice creation time
        "expires_at": unixtimestamp, // invoice expiration time
        "settled_at": unixtimestamp, // invoice settlement time, present when settled
        "cancel_reason": "string", // present when canceled
        "metadata": {}, // generic metadata, optional
        "commission": {
            "invoice": "string", // the commission invoice
            "amount": 2100000, // value in msats
            "state": "pending", // "pending", "settled" or "failed", present once the commission payment started
            "fees_paid": 0 // routing fees in msats, optional
        }
    }
}
```

The result object is the NIP-47 `lookup_invoice` object, extended by `cancel_reason` and `commission`.

A request repeating an earlier request on the same connection with identical parameters MUST return the original result.

`lookup_invoice` for a sale invoice returns the same object. On a commission connection, it MUST only return sale invoices created by that connection.

## Sale invoice states

Sale invoices use `pending`, `accepted`, `settled`, `expired` and `canceled` as defined in [09.md](09.md). `accepted` means the payer's payment is held while the commission payment is unresolved.

`cancel_reason` is one of:

- `failed`: The commission payment failed, for example because the client canceled its commission invoice.
- `timeout`: Not enough time was left to pay or settle the commission.
- `revoked`: The connection was revoked or expired.

## Validation

Before creating a sale invoice, the wallet service MUST decode the commission invoice and verify that:

1. its payment hash equals `payment_hash`
2. its amount `c` is set, `c <= floor(amount * max_rate_ppm / 1000000) + max_fixed_msat`, and `c` plus the maximum routing fee is less than `amount`
3. its payee is allowed by `payees`, if set
4. its expiry is at least 300 seconds after the sale invoice's expiry, since partial payments can be held for minutes before the commission payment starts
5. its network matches
6. the payment hash has never been seen by the wallet service, unless the request is a repeated request or the commission invoice belongs to the same wallet service
7. the budget can reserve `c` plus the maximum routing fee

A reservation is charged to the budget period in which the sale invoice was created, released if the sale invoice expires or is canceled, and replaced by the actual amounts once the commission is paid.

The wallet service MAY limit `expiry` and the number of `pending` or `accepted` sale invoices per connection.

## Timing

The sale invoice is the final hop of the payer's payment, so the commission is a new payment. The wallet service MUST itself provide the guarantees of a forwarding node, using:

- `M`: a safety margin in blocks, satisfying the `cltv_expiry_delta` requirements of [BOLT 2](https://github.com/lightning/bolts/blob/master/02-peer-protocol.md), at least the wallet service's own `cltv_expiry_delta`, and larger than the margin at which its backend fails back held HTLCs on its own
- `R_max`: the largest total CLTV of a commission route, including all hops, the final delta and any padding
- `T`: a tolerance for blocks mined while the payer's payment is in transit

The sale invoice's `min_final_cltv_expiry_delta` MUST be at least `R_max + M + T`. RECOMMENDED: `M = 80`, `T = 4`, `R_max = 244`, giving 328. It MUST stay well below the CLTV limits of payer wallets and intermediaries.

Outside that fail-back margin, the backend MUST NOT cancel a held sale invoice on its own while its commission payment is unresolved.

## Commission payment

When HTLCs for a sale invoice arrive, the wallet service:

1. MUST reject HTLCs expiring before the current block height plus `min_final_cltv_expiry_delta - T`.
2. Holds the HTLCs once they add up to `amount`, and MUST then reject further HTLCs for the payment hash. `E_in` is the lowest expiry of the held HTLCs.
3. Computes `L = min(E_in - M - current block height, R_max)`. If `L` is less than the commission invoice's final delta, it MUST cancel the sale with `timeout`.
4. Pays the commission invoice with a total CLTV limit of `L` per route attempt, including shadow route and final hop padding, and with routing fees capped as defined by the policy. It MUST NOT start more than one payment for the payment hash. Retries happen only within that payment.
5. On success, MUST verify the preimage, persist it, and settle the sale invoice without delay.
6. On failure, MUST cancel the sale invoice. The payment has failed only when no outgoing HTLC for the payment hash is unresolved. Until then, the sale invoice stays held, unresolved outgoing HTLCs are resolved on-chain like a forwarding node does, and a preimage revealed on-chain MUST be used to claim the incoming HTLCs before `E_in`.

Dust HTLCs and on-chain fee economics are handled as by any forwarding node. The user's exposure is then at most `c` per sale.

### Same wallet service

A commission invoice can be payable to the wallet service's own node, for example when the client and the user are both accounts of one custodial wallet service. Both invoices then carry the same payment hash on one node. Lightning nodes refuse a second invoice for a payment hash they already know, and the commission cannot be paid over the network.

Supporting this case is OPTIONAL. A wallet service that does not support it MUST reject such requests with `RESTRICTED`. Clients SHOULD then use a commission invoice from a different wallet.

A wallet service that supports it MUST pay the commission internally with the same semantics as a network payment: the client's account is credited only when the client settles its commission invoice, and the sale invoice is settled with that preimage. If the client has not settled or canceled by `E_in - M`, the wallet service MUST cancel both with `timeout`. It MUST match incoming payments to the sale invoice by `payment_secret` or invoice, not by payment hash, and clients SHOULD look up invoices by invoice string. If `payees` lists the wallet service's own node, the user MUST select the receiving account and only commission invoices of that account are paid. How the wallet service tracks two invoices with one payment hash, for example by intercepting HTLCs instead of using the node's invoice database, is outside the scope of this specification.

## Client behavior

The client:

1. Generates a 32-byte preimage, derives the payment hash, and persists both. It MUST NOT reuse a payment hash.
2. Calls `make_hold_invoice` on its own wallet for the commission, with an expiry at least 300 seconds after the sale invoice's and a `min_cltv_expiry_delta` small enough that the commission route, including it, fits in `R_max`.
3. Calls `make_commission_invoice`. After a timeout, it SHOULD repeat the identical request. After `RESTRICTED`, it SHOULD read the current policy with `get_info`. If the request fails definitively, it MUST cancel its commission invoice.
4. Verifies the sale invoice's payment hash, amount, expiry and payee, and shows it to the payer.
5. Waits for `hold_invoice_accepted` for its commission invoice. It MAY check with `lookup_invoice` that the sale invoice is `accepted`.
6. Either:

   * Calls `settle_hold_invoice` on its own wallet to capture the sale, or
   * Calls `cancel_hold_invoice` on its own wallet to void it.

The client MUST settle or cancel before the commission invoice's `settle_deadline`, and SHOULD do so promptly: a delayed client keeps the payer's funds locked for up to about `R_max` blocks, far longer than [03.md](03.md) intends.

Refunds after settlement are outside the scope of this specification.

## Notifications

`payment_received`, as defined in [02.md](02.md), SHOULD be sent when a sale invoice is settled. It contains the `make_commission_invoice` result object.

## Errors

- `RESTRICTED`: The connection has no commission permission, validation failed, or the commission invoice is payable to the wallet service's own node and that is not supported. The message MUST be identical for every validation failure, so that clients cannot learn which payment hashes the wallet service has seen.
- `QUOTA_EXCEEDED`: The budget cannot reserve the commission, or a limit on pending sale invoices is reached.

The wallet service can also return applicable NWC core errors.

## Security considerations

- The client knows the preimage before the payer pays. Two consequences: a client that is, or colludes with, a routing node on the payer's route can claim the payer's HTLC without forwarding it, so the user receives nothing; and the preimage is not proof of payment, so users and clients MUST rely on the wallet service's state, through `lookup_invoice` or `payment_received`, to learn that a sale was paid. Clients are trusted not to intercept. PTLCs would remove this risk.
- The client cannot spend from the user's wallet. A compromised client or leaked connection can at most create sale invoices, capture or cancel pending sales, read sale data and, without `payees`, redirect commissions within the policy.
- A client claiming late cannot make the user lose the sale, since the commission HTLC expires at least `M` blocks before the payer's HTLCs.
- Intermediaries on both routes learn the preimage when the client settles. Paying the commission only after the payer's HTLCs are held, rejecting later HTLCs, and never reusing a payment hash prevent them from claiming further HTLCs.
- Anyone can pay a commission invoice. Clients that need certainty that a sale exists check `lookup_invoice` before settling.
- Payee pinning by node pubkey has no effect when the payee node is shared, for example by hosted wallets.
- Both payments share a payment hash, so an observer on both routes can link them.

## Relationship to other specs

- NWC core defines `lookup_invoice`, `get_info` and the info event used for discovery.
- [08.md](08.md) is extended by the `commission_policy` parameter.
