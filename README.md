# Nostr Wallet Connect Specifications

This repository contains standalone Nostr Wallet Connect specifications for optional features that are intentionally kept out of the simplified NIP-47 core spec.

The core NWC protocol is tracked in NIP-47 for now. This repository reserves `01.md` for a future standalone core NWC spec equivalent to that simplified core, but starts with optional extensions from `02.md` onward.

## Specifications

| File          | Title | Description |
|---------------| --- | --- |
| [02](02.md)   | Notifications | Defines notification discovery, transport, the generic notification payload, and common notification types such as `payment_received` and `payment_sent`. |
| [03](03.md)   | Hold Invoices | Defines `make_hold_invoice`, `cancel_hold_invoice`, `settle_hold_invoice`, and the hold-invoice lifecycle. |
| [04](04.md)   | Keysend Payments | Defines the `pay_keysend` method for optional keysend support. |
| [05](05.md)   | Transaction History | Defines the `list_transactions` method for optional transaction history listing. |
| [06](06.md)   | Metadata Conventions | Defines common metadata keys and limits used by optional NWC features. |
| [07](07.md)   | Deep Links | Defines optional mobile deep-link conventions for NWC pairing flows. |
| [09](09.md)   | Payment Lookup | Defines the generic `lookup_payment` method and common payment record envelope. |
| [12](12.md)   | BOLT12 Offers | Defines `make_offer` and BOLT12 payment details for `lookup_payment`. |
| [321](321.md) | BIP-321 Lightning Payments | Defines the `pay` and `receive` methods for BIP-321 Lightning payment instructions. |
