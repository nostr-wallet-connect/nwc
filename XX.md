NWC-XX
======

Ark Payments
------------

`draft` `optional`

## Summary

This specification adds off-chain [Ark](https://ark-protocol.org) payments to Nostr Wallet Connect. It adds no new methods. It defines:

- the `ark` payment instruction for the BIP-321 `pay` and `receive` methods in [321.md](321.md)
- the `ark` payment type, its selectors and its `details` object for `lookup_payment` in [09.md](09.md), which are also used by [05.md](05.md) and [02.md](02.md)

## Motivation

Ark wallets such as Bark, Arkade and Wavelength hold bitcoin off-chain as VTXOs on an Ark server. Two users of the same Ark server can pay each other directly. The server cosigns the payment, which takes effect in seconds, with no round, on-chain transaction or Lightning route. The recipient does not need to be online: the server holds the payment until the recipient's wallet collects it.

An Ark payment is made to an Ark address. The address has no amount and no expiry, and can be paid more than once. It is a bech32m string, so [BIP-321](https://github.com/bitcoin/bips/blob/master/bip-0321.mediawiki) already gives it a query key: "New payment instructions using bech32 or bech32m encodings SHOULD reuse their address format's Human Readable Part as the parameter key." An Ark address commits to its Ark server, so a wallet can tell from the address alone whether it can pay it off-chain. If it cannot, it pays another instruction in the same URI, usually Lightning.

NWC-321 already lets a client hand the wallet a URI and lets the wallet pick the instruction. This specification only defines what is specific to Ark: the instruction and the payment record.

## Dependencies

Implementations use:

- [321.md](321.md) for the `pay` and `receive` methods
- [09.md](09.md) for `lookup_payment` and the common payment record envelope
- [BOAT-0001](https://github.com/ark-protocol/boats/blob/master/boat-0001.md) for the Ark address encoding

## Terms

- **Ark server**: the operator that cosigns a wallet's off-chain transactions. A wallet uses one Ark server at a time. The server is identified by its 33-byte compressed secp256k1 public key, encoded as lowercase hex.
- **Ark address**: a BOAT-0001 address. It uses bech32m with human-readable part `ark` on mainnet or `tark` on test networks, followed by a version and a version-specific payload. It commits to one Ark server:
    - For a version 0 address, the address `server_key` is the x-only form of the server key.
    - For a version 1 address, the address `ark_id` is the first 4 bytes of SHA-256 of the server key.
- **VTXO outpoint**: the `<txid>:<vout>` identifier of a VTXO. The txid is written as 64 lowercase hex characters and the vout in decimal.

An off-chain Ark payment can only be made between two users of the same Ark server.

## Discovery

A wallet service that supports this specification MUST:

- support `pay` from [321.md](321.md) with `ark` instructions, and support `lookup_payment` from [09.md](09.md) with the `ark` payment type
- list `pay` and `lookup_payment` in its NWC info event and `get_info` `methods`, for the connections allowed to call them
- include `321`, `09` and `XX` in the `extensions` tag of its NWC info event (kind `13194`), and in the `extensions` array of its `get_info` result

A wallet service that also implements `receive` SHOULD include `ark` instructions as described below.

Example info event:

```yaml
{
    "kind": 13194,
    "tags": [
        ["encryption", "nip44_v2"],
        ["extensions", "02 09 321 XX"],
        ["notifications", "payment_received payment_sent"]
    ],
    "content": "get_info get_balance pay_invoice make_invoice lookup_invoice pay receive lookup_payment"
}
```

## The `ark` payment instruction

### Encoding

An Ark payment instruction is a BIP-321 query parameter. Its key is the address's human-readable part, as BIP-321 requires: `ark` for a mainnet address and `tark` for a test-network address.

```
bitcoin:?ark=ark1pwh9vsmezqqpjy9akejayl2vvcse6he97rn40g84xrlvrlnhayuuyefrp9nse2yspqqjl5wpy&lightning=lnbc...
```

Existing wallets also put test-network addresses under the `ark` key. A wallet service MUST accept either key with either kind of address, and MUST check the network from the address itself.

A URI MAY contain more than one Ark instruction, for example one for each Ark server the recipient uses. BIP-321 allows repeated payment-instruction keys.

### `pay`

When the URI contains Ark instructions, the wallet service selects one as follows:

- It MUST select an Ark instruction only if the address is valid for the wallet's network and commits to the wallet's Ark server.
- It MUST also be able to pay the address's version and contents, for example its VTXO policy and delivery mechanism.
- If several Ark instructions qualify, it selects one of them.
- If no Ark instruction qualifies, it MUST NOT pay any of them. It selects another supported instruction from the URI, as [321.md](321.md) describes.
- If no instruction in the URI can be paid, it returns `UNSUPPORTED_PAYMENT_INSTRUCTION` before moving any funds.

A wallet service SHOULD prefer a qualifying Ark instruction over Lightning instructions in the same URI. It pays the recipient directly and avoids a Lightning fee.

Ark addresses carry no amount and Ark payments move whole satoshis. The `amount` comes from the request or from the URI `amount`. If the selected instruction is `ark` and that amount is not a positive multiple of 1000 msats, the wallet service MUST return `BAD_REQUEST`. It MUST NOT round the amount.

An Ark payment carries no message to the recipient. If `payer_note` is not empty, the wallet service MUST NOT select an Ark instruction.

This specification defines no BIP-321 proof-of-payment information for Ark payments. The wallet service MUST NOT select an Ark instruction from a URI that contains `req-pop`.

`max_fee` limits the fee the wallet pays to its Ark server, as it limits routing fees for Lightning.

Response:

```yaml
{
    "result_type": "pay",
    "result": {
        "transaction_id": "string", // wallet-scoped transaction identifier
        "state": "settled", // "pending", "settled", or "failed"
        "instruction_type": "ark",
        "amount": 21000, // amount the recipient receives, in msats
        "fees_paid": 0, // fee paid to the Ark server, in msats
        "failure_reason": "string", // optional unless state is "failed"
        "created_at": unixtimestamp,
        "settled_at": unixtimestamp // optional unless state is "settled"
    }
}
```

`instruction_type` is `ark` for both the `ark` and `tark` keys. An Ark payment has no `payment_hash`, `preimage`, `payer_proof` or `txid`, so these fields are absent.

The wallet service MAY respond while the payment is still `pending`. The client then calls `lookup_payment` with the `transaction_id` until the state is final.

### `receive`

A wallet service that supports this specification SHOULD include an Ark instruction in the URI returned by `receive`, next to its Lightning instructions:

```yaml
{
    "result_type": "receive",
    "result": {
        "bip321": "bitcoin:?lightning=lnbc...&ark=ark1p...", // BIP-321 URI
        "transaction_id": "string" // optional
    }
}
```

The Ark address MUST be for the wallet's current Ark server and network. The wallet service SHOULD return an address it has not returned before. This keeps payments to different requests from being linked, and lets the wallet tell which request a payment was for.

An Ark address has no amount and no expiry. When `amount` is set, the URI carries it in the BIP-321 `amount` field. Even so, the address itself can be paid any amount, any number of times. The wallet MUST keep accepting payments to every address it has returned for as long as it uses the same Ark server.

Like a BOLT12 offer, an Ark address is a receive target, not a payment record. Each payment received on it is its own record.

## The `ark` payment type

This specification defines `ark` as a payment type for `lookup_payment` in [09.md](09.md).

### Lookup selectors

An Ark payment can always be looked up by `transaction_id`. It can also be looked up by address or by VTXO outpoint:

```yaml
{
    "method": "lookup_payment",
    "params": {
        "payment_type": "ark",
        "lookup": {
            "vtxo": "d9b1d0e6ea53ab1400edc210615cb1b8b1fa67edfa0fd3db876209bfe28e6066:0"
        }
    }
}
```

The `lookup` object MUST use exactly one of:

- `vtxo`: a VTXO outpoint. It matches the payment that created this VTXO for the recipient. A recipient wallet uses it to match a payment that the sender has reported.
- `address`: an Ark address. It matches incoming payments received on the address, or outgoing payments made to it.

An address can be paid more than once. If a selector matches more than one record visible to the connection, the wallet service MUST return `MULTIPLE_MATCHES`.

### Payment details

A record with `payment_type` set to `ark` uses the common envelope from [09.md](09.md) with this `details` object:

```yaml
{
    "address": "ark1p...", // Ark address paid or received on
    "ark_server_pubkey": "02...", // Ark server the payment was made on, 33-byte compressed key in hex
    "vtxos": ["<txid>:<vout>"] // VTXOs the payment created for the recipient, optional
}
```

The fields are:

- `address`: REQUIRED for outgoing payments. For incoming payments it SHOULD be returned when the wallet knows which of its addresses was paid.
- `ark_server_pubkey`: REQUIRED.
- `vtxos`: the outpoints of the VTXOs that the payment created for the recipient. It does not include change returned to the sender. A recipient wallet SHOULD return it. A sender wallet SHOULD return it when it knows the outpoints.

The common `type` field gives the direction. The common `amount` is the amount the recipient receives, in millisatoshis (satoshis × 1000). The common `fees_paid` is the fee the paying wallet paid to its Ark server.

Ark payment records never contain a `payment_hash`. This holds for records returned by `pay`, `lookup_payment` and `list_transactions`, and for records in notifications. Clients MUST NOT assume a `payment_hash` when `payment_type` is `ark`.

### States

The applicable states are `pending`, `settled` and `failed`:

- `pending`: the payment has started, but the Ark server has not yet cosigned it, or the recipient's wallet does not yet hold it.
- `settled`: the Ark server has cosigned the payment and the recipient can spend the VTXOs. For an incoming payment, the wallet holds the received VTXOs and can spend them.
- `failed`: an outgoing payment ended without reaching the recipient.

`settled` does not mean confirmed on-chain. Like other off-chain Ark funds, received VTXOs depend on the Ark server until the recipient's wallet refreshes them into a batch. Expiry and refresh are wallet maintenance and are outside this specification.

### Transaction history and notifications

When a wallet supports [05.md](05.md), every Ark payment returned by `list_transactions` MUST use the common envelope and the `details` object above.

When a wallet supports [02.md](02.md), it SHOULD send `payment_received` when an incoming Ark payment becomes `settled`, and `payment_sent` when an outgoing one does. The `notification` object is the same record that `lookup_payment` returns, with `payment_type: "ark"`.

Ark payments to a published address arrive without any request from the client. Notifications are how a client learns about them without polling.

## Example

A client pays a recipient who uses an Ark wallet and also has a Lightning address. The recipient's two Ark addresses are on two different Ark servers. One is a version 0 address, as used by Arkade. The other is a version 1 address, as used by Bark.

```yaml
{
    "method": "pay",
    "params": {
        "payment": "bitcoin:?ark=ark1qzpq904am6clw3pgqwyh4p02708fy4xs0hcpwt7rwfdttuxsjameetl3ujgrw8089sl27rtp79aqcl0xspkahwnm4teg5lmhe47pxulw9m6rn8&ark=ark1pwh9vsmezqqpjy9akejayl2vvcse6he97rn40g84xrlvrlnhayuuyefrp9nse2yspqqjl5wpy&lightning=lnbc210n1...",
        "amount": 21000
    }
}
```

1. The wallet's Ark server is `038202bebddeb1f7442803897a85eaf3ce9254d07df0172fc3725ab5f0d097779c`. The version 0 address commits to its x-only form, so the wallet selects that instruction.
2. The wallet pays the address off-chain and responds with `instruction_type: "ark"` and a `transaction_id`.
3. The client calls `lookup_payment` with the `transaction_id`. `details.vtxos` holds the outpoints of the recipient's new VTXOs.
4. A wallet on any other Ark server finds no matching Ark instruction. It pays the Lightning invoice instead and reports `instruction_type: "bolt11"`.

## Security and privacy considerations

- **Address reuse.** An Ark address can be paid many times, and a published address links every payment made to it. Wallets SHOULD return fresh addresses from `receive`.
- **Server visibility.** The Ark server sees the sender's and recipient's VTXOs for every payment it cosigns.
- **Retries.** An Ark payment cannot be refused or deduplicated by the recipient. Unlike a Lightning invoice, a URI with only an Ark instruction is paid again if the request is retried. A client SHOULD check with `lookup_payment` or `list_transactions` before retrying a `pay` request whose response it did not receive. A generic idempotency mechanism for `pay` would solve this for every instruction type. It is outside this specification.
- **Budgets.** Connection budgets, such as those in [13.md](13.md), apply to Ark payments. Both `amount` and `fees_paid` count toward the budget.
- **Settlement.** A client SHOULD treat a payment as received only once it is `settled`.

## Relationship to other specs

- [321.md](321.md) defines `pay` and `receive`. This specification adds the `ark` instruction and the `instruction_type` value `ark`.
- [09.md](09.md) defines `lookup_payment` and the common payment record envelope. This specification defines the `ark` payment type.
- [05.md](05.md) defines `list_transactions`. Ark items use the same record shape as `lookup_payment`.
- [02.md](02.md) defines payment notifications. Ark notifications carry the same record.
- [13.md](13.md) defines connection budgets, which apply to Ark payments.
