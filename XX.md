NWC-XX
======

Logout
------

`draft` `optional`

## Summary

This specification defines the optional `logout` method, which lets a client revoke its own Nostr Wallet Connect connection.

It builds on the request, response, and discovery mechanisms in [NIP-47](https://github.com/nostr-protocol/nips/blob/master/47.md).

## Motivation

When a user logs out of a client or disconnects their wallet, the client discards the connection URI, but the connection stays authorized on the wallet service. Its secret can still spend from the wallet if it was copied, backed up, or leaked, and it lingers in the user's list of connections until they revoke it by hand.

`logout` lets the client revoke the connection at the moment the user stops using it.

Revoking a connection is a convenience rather than a requirement for interoperability: a wallet service without `logout` answers `NOT_IMPLEMENTED`, and the user can still revoke the connection in their wallet. It is therefore specified separately from the NWC core protocol.

## Discovery and permissions

A wallet service supporting this specification MUST advertise `logout` in its NWC info event and in the `get_info` response's `methods` array.

The wallet service SHOULD include `XX` in the `extensions` tag of its NWC info event (kind `13194`) and in the `extensions` array of the `get_info` response's `result` object.

Every connection is implicitly granted `logout`, whatever other permissions it has. Because `logout` is implicit, it does not count as a granted request method under [NWC-08](08.md) and does not have to be listed in `request_methods` or `optional_request_methods`.

## Methods

### `logout`

Revokes the connection used to send the request. The method takes no parameters.

Request:

```json
{
    "method": "logout",
    "params": {}
}
```

Response:

```json
{
    "result_type": "logout",
    "result": {}
}
```

The wallet service MUST durably revoke the connection before responding successfully, so that it never confirms a `logout` that did not take effect. Once the connection is revoked, any request other than `logout` signed by its key MUST be rejected, regardless of the request's `created_at`, since a holder of a leaked key can backdate it. Revoking also covers everything else the wallet service sends for the connection: it MUST NOT send [02](02.md) notifications to the connection's public key anymore, and MUST delete any NWC info event (kind `13194`) it published for the connection or stop refreshing it. Whether the connection is deleted or kept in a disabled state is up to the wallet service.

A payment that was already in flight when the connection was revoked MAY still complete. A wallet service SHOULD keep a revoked connection recognizable and answer a repeated `logout` on it with success again, which makes the method idempotent, while every other request signed by the connection's key SHOULD be answered with an `UNAUTHORIZED` error.

Isolated connections (see `isolated` in [NWC-08](08.md)) can hold a balance of their own. Revoking such a connection MUST NOT make that balance unreachable: the wallet service MUST keep the funds available to the user after the connection is revoked. A wallet service that cannot do that MAY decline the request with an applicable error such as `OTHER` instead, leaving the connection usable.

Clients SHOULD send `logout` when the user logs out or disconnects their wallet. Clients SHOULD discard the connection URI whether or not the request succeeds, and SHOULD NOT let a missing or failed response block the user from logging out. A client SHOULD treat an `UNAUTHORIZED` response to `logout` as a successful logout.

## Errors

The wallet service uses applicable NWC core errors, including `NOT_IMPLEMENTED` when the method is unsupported.
