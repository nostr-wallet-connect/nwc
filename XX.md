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

Every connection MUST be allowed to call `logout`, whatever other permissions it has.

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

After responding successfully, the wallet service MUST NOT honor further requests signed by the connection's key, and SHOULD respond to them with an `UNAUTHORIZED` error.

Clients SHOULD send `logout` when the user logs out or disconnects their wallet. Clients SHOULD discard the connection URI whether or not the request succeeds, and SHOULD NOT let a missing or failed response block the user from logging out.

## Errors

The wallet service uses applicable NWC core errors, including `NOT_IMPLEMENTED` when the method is unsupported.
