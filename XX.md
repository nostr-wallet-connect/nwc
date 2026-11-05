NWC-XX
======

Push Notifications
------------------

`draft` `optional`

## Summary

This specification defines the optional `register_push` and `unregister_push` methods. A client uses them to ask the wallet service to deliver its [NWC-02](02.md) notifications to a push endpoint, in addition to the usual relay events.

Delivery uses the Web Push protocol ([RFC 8030](https://www.rfc-editor.org/rfc/rfc8030)) with message encryption ([RFC 8291](https://www.rfc-editor.org/rfc/rfc8291)) and VAPID ([RFC 8292](https://www.rfc-editor.org/rfc/rfc8292)). That one protocol reaches [UnifiedPush](https://unifiedpush.org) distributors, browser push services, and gateways that app developers run in front of Firebase Cloud Messaging or Apple Push Notification service.

## Motivation

Mobile operating systems close a background app's relay connections, so a client misses [NWC-02](02.md) notifications while it is not in the foreground.

Today this is solved with a bridge run by the app developer: the client registers its push token with the bridge, and the bridge keeps a relay subscription for every connection and forwards each notification event through its push provider. The bridge is one more service to run, it learns which connections are active and when they receive payments, and it only serves clients of that developer.

With this specification the wallet service, which already knows when a notification happens, delivers it directly:

- no bridge and no relay subscription per device;
- the wallet service needs no push provider credentials: the client brings an endpoint and keys, and any Web Push endpoint works;
- the push service only sees ciphertext, encrypted to keys that only the client holds;
- the registration travels inside the NWC connection, so no other account, URL or credential is needed.

## Discovery and permissions

A wallet service supporting this specification MUST advertise `register_push` and `unregister_push` in its NWC info event, and MUST include them in the `get_info` response's `methods` array when the connection may use them.

The wallet service SHOULD include `XX` in the `extensions` tag of its NWC info event (kind `13194`) and in the `extensions` array of the `get_info` response's `result` object.

A connection MAY use these methods only when it is authorized to receive notifications. Push delivery never widens what a connection can see: a registered endpoint receives exactly the notifications the connection would receive as relay events.

When the connection may use these methods, the `get_info` response MUST include the wallet service's VAPID public key, which browsers and UnifiedPush distributors need before they create an endpoint:

```yaml
{
    "result_type": "get_info",
    "result": {
        "push_vapid_key": "BOwX..." // uncompressed P-256 public key, base64url without padding
    }
}
```

The wallet service SHOULD keep this key stable. A client whose endpoint was created for another key MUST create a new endpoint and register it again.

## Methods

### `register_push`

Registers a push endpoint for the requesting connection. Registering an endpoint that is already registered for this connection replaces its keys and notification types.

Request:

```yaml
{
    "method": "register_push",
    "params": {
        "endpoint": "https://ntfy.sh/upYm9Ft3v1?up=1", // push endpoint, https
        "p256dh": "BCVxsr7N_eNgVRqvHtD0zTZsEc6...", // client's P-256 public key (RFC 8291 ua_public), base64url
        "auth": "BTBZMqHH6r4Tts7J_aSIgg", // 16-byte authentication secret (RFC 8291), base64url
        "notification_types": ["payment_received"] // optional, defaults to every type the connection receives
    }
}
```

Response:

```yaml
{
    "result_type": "register_push",
    "result": {
        "notification_types": ["payment_received"] // the types that will be pushed
    }
}
```

- `endpoint`: REQUIRED. MUST use `https`. The wallet service SHOULD refuse endpoints on loopback, private or link-local addresses with `BAD_REQUEST`.
- `p256dh` and `auth`: REQUIRED. Values as produced by `PushSubscription.toJSON().keys` in browsers and by UnifiedPush connector libraries.
- `notification_types`: OPTIONAL. The wallet service MUST ignore types the connection is not authorized for, and returns the resulting list.

The wallet service MAY limit the number of endpoints per connection and returns `QUOTA_EXCEEDED` when the limit is reached.

### `unregister_push`

Removes a push endpoint of the requesting connection. The method succeeds when the endpoint is not registered.

Request:

```yaml
{
    "method": "unregister_push",
    "params": {
        "endpoint": "https://ntfy.sh/upYm9Ft3v1?up=1"
    }
}
```

Response:

```json
{
    "result_type": "unregister_push",
    "result": {}
}
```

## Delivery

For every notification it publishes to the connection, the wallet service sends one push message to each matching endpoint of that connection:

- an HTTP `POST` to the endpoint with the encrypted payload as the body, `Content-Encoding: aes128gcm` (RFC 8291), a `TTL` header, and an `Authorization: vapid` header signed with the key advertised in `push_vapid_key` (RFC 8292);
- `Urgency: high` for `payment_received` and for notifications that need an action, such as `hold_invoice_accepted` from [NWC-03](03.md); `normal` otherwise.

The wallet service MUST still publish the notification events to relays as defined in [NWC-02](02.md). Push is an additional path.

### Payload

The plaintext is the [NWC-02](02.md) notification payload with the id of the notification event the wallet service published:

```yaml
{
    "event_id": "5f1c...", // id of the kind 23197 event, or 23196 when only NIP-04 is used
    "notification_type": "payment_received",
    "notification": {
        "type": "incoming",
        "payment_hash": "0123456789abcdef...",
        "amount": 21000
        // ...the rest of the NWC-02 notification
    }
}
```

A push message carries at most 4096 bytes. When the encrypted payload would be larger, the wallet service MUST reduce `notification` to `type`, `state`, `payment_hash` and `amount`, and add `"truncated": true`. The client then reads the full payment with `lookup_payment` from [NWC-09](09.md).

Notification events are ephemeral (kinds `23196` and `23197`), so relays do not keep them and `event_id` cannot be used to fetch one later. Clients SHOULD use `event_id` to ignore a notification that arrives both by push and by relay.

### Endpoint lifecycle

- When the push service answers `404` or `410`, the wallet service MUST remove the endpoint.
- On `429` and `5xx` answers or network errors, the wallet service SHOULD retry with backoff for a limited time.
- When a connection is deleted or expires, the wallet service MUST remove its endpoints.
- Endpoints can change. Clients SHOULD call `register_push` when they start and whenever their endpoint or keys change, and `unregister_push` before discarding an endpoint.

## Errors

The wallet service uses applicable NWC core errors, including:

- `NOT_IMPLEMENTED` when the methods are unsupported;
- `RESTRICTED` when the connection is not authorized to receive notifications;
- `QUOTA_EXCEEDED` when the connection has too many endpoints;
- `BAD_REQUEST` when a parameter, the endpoint or the keys are invalid.

## Security and privacy considerations

- The registration is authenticated and encrypted by the NWC request itself. Only the holder of the connection secret can add or remove endpoints for that connection, and the authentication secret never leaves the encrypted request.
- Only the wallet service and the client know the RFC 8291 keys, so the push service can neither read nor forge payloads.
- The push service learns the endpoint, the time and the size of each message. Clients that want to avoid a large push provider can use a UnifiedPush distributor with a self-hosted server. Wallet services MAY pad payloads to a fixed size.
- The wallet service learns which push service the client uses.
- A wallet service sends HTTP requests to client-supplied URLs. It MUST NOT follow redirects, and SHOULD refuse local and private addresses and limit request time and response size.

## Implementation notes

- **UnifiedPush (Android):** connector libraries return the endpoint and RFC 8291 keys after registration with `push_vapid_key` as the VAPID key.
- **Browsers and web apps:** `PushManager.subscribe({userVisibleOnly: true, applicationServerKey: push_vapid_key})` returns the endpoint and keys.
- **Firebase Cloud Messaging and Apple Push Notification service:** these do not accept Web Push messages from third parties. An app developer can run a small gateway that exposes a Web Push endpoint per device and relays the still-encrypted body to the device through its provider, so the gateway cannot read it.

## References

- [NWC-02](02.md): notification types and payload.
- [RFC 8030](https://www.rfc-editor.org/rfc/rfc8030), [RFC 8291](https://www.rfc-editor.org/rfc/rfc8291), [RFC 8292](https://www.rfc-editor.org/rfc/rfc8292): Web Push delivery, encryption and VAPID.
- [UnifiedPush](https://unifiedpush.org): open push for Android.
