NWC-XX
======

Ark Payments
------------

`draft` `optional`

## Summary

This specification defines optional Nostr Wallet Connect support for paying and receiving off-chain [Ark](https://ark-protocol.org) payments to and from Ark addresses.

It includes:

- `make_ark_address`, which returns an Ark address that the wallet receives payments on
- `pay_ark_address`, which pays an Ark address off-chain
- the `ark` object in `get_info`, which describes the wallet's Ark server
- the `ark` payment type, selectors and `details` object for `lookup_payment` in [09.md](09.md)

## Motivation

Ark wallets such as bark, Arkade and Wavelength hold bitcoin off-chain as VTXOs on an Ark server. Two users of the same Ark server can pay each other directly: the payment is cosigned by the server and takes effect in seconds, without a round, an on-chain transaction or a Lightning route.

These wallets already expose Lightning through NWC core. But an Ark payment is not a Lightning payment:

- It is paid to an address, not to an invoice. The address is reusable, has no amount and no expiry, and can be published in advance, for example in a profile.
- The recipient does not need to be online when the payment is made. The Ark server holds the payment until the recipient's wallet collects it.
- There is no payment hash or preimage. Paying the same address twice makes two payments.

Paying another user of the same Ark server over Lightning instead costs a Lightning fee and a liquidity round-trip through the server. It also requires the recipient to be online to create an invoice.

These methods are separate from the NWC core because only Ark wallets can support them.

## Dependencies

Implementations use:

- [NIP-47](https://github.com/nostr-protocol/nips/blob/master/47.md) for the core protocol, which allows extensions to define payment rails other than Lightning
- [09.md](09.md) for the generic `lookup_payment` method and common payment record envelope
- [06.md](06.md) for metadata conventions
- [BOAT-0001](https://github.com/ark-protocol/boats/blob/master/boat-0001.md) for the Ark address encoding

## Terms

- **Ark server**: the operator that cosigns the wallet's off-chain transactions. A wallet uses one Ark server at a time.
- **Ark address**: a [BOAT-0001](https://github.com/ark-protocol/boats/blob/master/boat-0001.md) address, a bech32m string with human-readable part `ark` for mainnet or `tark` for test networks, followed by a version and a version-specific payload. It identifies a recipient on one Ark server.

An off-chain Ark payment can only be made between two users of the same Ark server. This specification does not define payments between Ark servers. Those are made over Lightning.

## Discovery

A wallet service supporting this specification MUST:

- advertise `make_ark_address`, `pay_ark_address` and `lookup_payment` as supported methods in its NWC info event and `get_info` response, for the methods the connection is authorized to call
- include `09` and `XX` in the `extensions` tag of its NWC info event (kind `13194`) and in the `extensions` array of its `get_info` result
- support `ark` as a `payment_type` for `lookup_payment`
- include the `ark` object in its `get_info` response

Example info event:

```yaml
{
    "kind": 13194,
    "tags": [
        ["encryption", "nip44_v2"],
        ["extensions", "02 09 XX"],
        ["notifications", "payment_received payment_sent"]
    ],
    "content": "get_info get_balance pay_invoice make_invoice lookup_invoice make_ark_address pay_ark_address lookup_payment"
}
```

A connection MAY be authorized for `make_ark_address` without `pay_ark_address`, for example a receive-only connection.

### `get_info`

The `get_info` result gains an `ark` object:

```yaml
{
    "result_type": "get_info",
    "result": {
        "network": "mainnet",
        "methods": ["get_info", "make_ark_address", "pay_ark_address", "lookup_payment"],
        "extensions": ["09", "XX"],
        // ... other get_info fields
        "ark": {
            "server_pubkey": "02...", // the Ark server's 33-byte compressed public key, lowercase hex
            "server_url": "https://ark.example.com" // the Ark server's URL, optional
        }
    }
}
```

`server_pubkey` is the key that BOAT-0001 addresses commit to. It lets a client tell, without contacting the wallet again, whether an address is for this wallet's Ark server:

- For a version 0 address, the address `server_key` equals the x-only form of `server_pubkey`.
- For a version 1 address, the address `ark_id` equals the first 4 bytes of SHA-256 of `server_pubkey`.

This check is a hint for choosing between Ark and Lightning. The wallet service makes the authoritative check when it is asked to pay.

`network` uses the values of NWC core. A `mainnet` wallet uses `ark` addresses and every other network uses `tark` addresses.

## Methods

### `make_ark_address`

Returns an Ark address that the wallet can be paid on.

Request:

```yaml
{
    "method": "make_ark_address",
    "params": {
        "metadata": {} // optional metadata as defined in 06.md
    }
}
```

Response:

```yaml
{
    "result_type": "make_ark_address",
    "result": {
        "address": "ark1p...", // BOAT-0001 Ark address
        "created_at": unixtimestamp
    }
}
```

The address MUST be for the wallet's current Ark server and network.

The wallet SHOULD return an address it has not returned before. A fresh address improves privacy and lets the wallet attribute incoming payments to the request that created the address.

An Ark address has no amount and no expiry, and it can be paid more than once. The wallet MUST keep accepting payments to every address it has returned for as long as it uses the same Ark server.

The wallet SHOULD associate request `metadata` with payments received on the returned address. When it does, that metadata is returned in the common `metadata` field of the corresponding `lookup_payment` record.

Like a BOLT12 offer, an address is a receive target, not a payment record. Calling `make_ark_address` does not create a record that `lookup_payment` can return.

Errors:

- `TEMPORARILY_UNAVAILABLE`: The wallet cannot create an address now, for example because it cannot reach its Ark server.

The wallet service can also return applicable NWC core errors.

### `pay_ark_address`

Pays an Ark address with an off-chain Ark payment.

Request:

```yaml
{
    "method": "pay_ark_address",
    "params": {
        "address": "ark1p...", // BOAT-0001 Ark address, required
        "amount": 21000000, // amount the recipient receives, in msats, required
        "idempotency_key": "string", // client-chosen key that makes retries safe, optional
        "metadata": {} // optional metadata as defined in 06.md
    }
}
```

`amount` is the amount the recipient receives. Ark payments move whole satoshis. If `amount` is not a positive multiple of 1000, the wallet MUST reject the request with `BAD_REQUEST`. It MUST NOT round the amount.

Fees are paid on top of `amount`.

`metadata` is stored by the paying wallet only. An Ark payment carries no message to the recipient.

Before moving any funds, the wallet service MUST check that:

- the address is a valid Ark address
- the address is for the wallet's network
- the address is for the wallet's Ark server, and uses an address version, VTXO policy and delivery mechanism that the wallet can pay

If any check fails, the wallet MUST return an error and MUST NOT make the payment.

#### Retries

A Lightning payment is safe to retry, because the payment hash stops a second payment. An Ark payment has no such protection. A client that may retry a request, for example after a timeout or reconnect, SHOULD set `idempotency_key`.

When `idempotency_key` is present:

- The wallet MUST make at most one payment for each key on a connection.
- If the key was already used on this connection for the same `address` and `amount`, the wallet MUST NOT pay again. It MUST return the result of the existing payment, which may still be `pending`.
- If the key was already used on this connection with a different `address` or `amount`, the wallet MUST return `BAD_REQUEST`.
- The wallet MUST remember keys for at least as long as it keeps the payment record, including across restarts.

Response:

```yaml
{
    "result_type": "pay_ark_address",
    "result": {
        "transaction_id": "string", // wallet-scoped transaction identifier
        "state": "settled", // "pending", "settled", or "failed"
        "amount": 21000000, // amount the recipient receives, in msats
        "fees_paid": 0, // fees in msats, optional if unavailable
        "failure_reason": "string", // optional unless state is "failed"
        "created_at": unixtimestamp,
        "settled_at": unixtimestamp // optional unless state is "settled"
    }
}
```

The `transaction_id` MUST identify the same payment in `lookup_payment`, in `list_transactions` and in notifications.

A wallet MAY respond while the payment is still `pending`, for example when the Ark server is slow to respond. The client then uses `lookup_payment` to learn the outcome.

Errors:

- `BAD_REQUEST`: The address is not a valid Ark address, the amount is invalid or below the Ark server's minimum, or the `idempotency_key` was used for a different payment.
- `UNSUPPORTED_NETWORK`: The address is for a different Bitcoin network.
- `UNSUPPORTED_ARK_ADDRESS`: The wallet cannot pay this address off-chain. This happens when the address is for another Ark server, or uses an address version, VTXO policy or delivery mechanism that the wallet does not support. No funds were moved. The client MAY pay the recipient over Lightning instead.
- `PAYMENT_FAILED`: The wallet attempted the payment, but it failed.

The wallet service can also return applicable NWC core errors, such as `INSUFFICIENT_BALANCE`, `QUOTA_EXCEEDED` and `RESTRICTED`.

### `lookup_payment`

This specification defines `ark` as a payment type for the generic method in [09.md](09.md).

In addition to generic lookup by `transaction_id`, an Ark payment can be looked up by idempotency key or by address.

Request by idempotency key:

```yaml
{
    "method": "lookup_payment",
    "params": {
        "payment_type": "ark",
        "lookup": {
            "idempotency_key": "string"
        }
    }
}
```

The `lookup` object MUST use exactly one of these selector forms:

- `idempotency_key`: the key passed to `pay_ark_address` on this connection. It matches the outgoing payment made with that key.
- `address`: an Ark address. It matches incoming payments received on the address, or outgoing payments made to it.

An address can be paid more than once, so `address` alone is not guaranteed to identify one payment. If a selector matches multiple wallet-visible records, the wallet MUST return `MULTIPLE_MATCHES`. Clients SHOULD keep the `transaction_id` for later lookup.

Response:

```yaml
{
    "result_type": "lookup_payment",
    "result": {
        "transaction_id": "wallet-scoped-id",
        "type": "outgoing", // "incoming" or "outgoing"
        "state": "settled", // "pending", "settled", or "failed"
        "payment_type": "ark",
        "amount": 21000000, // amount the recipient received, in msats
        "fees_paid": 0, // fees in msats, optional
        "created_at": unixtimestamp,
        "updated_at": unixtimestamp, // optional
        "settled_at": unixtimestamp, // optional unless state is "settled"
        "failure_reason": "string", // optional unless state is "failed"
        "metadata": {}, // optional metadata as defined in 06.md
        "details": {
            "address": "ark1p...", // Ark address paid or received on, optional for incoming payments
            "idempotency_key": "string", // key passed to pay_ark_address, optional
            "vtxos": ["txid:vout"] // VTXOs created for the recipient, optional
        }
    }
}
```

The common fields follow [09.md](09.md). Within `details`:

- `address` is REQUIRED for outgoing payments. For incoming payments it SHOULD be returned when the wallet can tell which of its addresses was paid.
- `idempotency_key` MUST be returned only to the connection that set it.
- `vtxos` lists the outpoints of the VTXOs that the payment created for the recipient, when the wallet knows them.

## Payment states and amounts

The applicable states for `ark` records are `pending`, `settled` and `failed`. Ark addresses do not expire and cannot be canceled, so `expired`, `canceled` and `accepted` do not apply.

- `pending`: the wallet has started the payment, but the Ark server has not yet cosigned it or the payment has not yet been made available to the recipient.
- `settled`: for an outgoing payment, the Ark server has cosigned it and it is available to the recipient. For an incoming payment, the wallet holds the received VTXOs and can spend them.
- `failed`: an outgoing payment ended without reaching the recipient. `failure_reason` explains why.

`settled` means final under the wallet's policy, as defined in [09.md](09.md). It does not mean confirmed on-chain. Like other off-chain Ark VTXOs, a received payment depends on the Ark server until the recipient's wallet refreshes it into a batch. Expiry and refresh are wallet maintenance and are not exposed by this specification.

The common `amount` is the amount the recipient receives, in millisatoshis (satoshis × 1000). The optional common `fees_paid` is the fee the paying wallet paid to the Ark server for this payment, in millisatoshis. It is absent or zero for incoming payments.

## Transaction history

When a wallet supports both this specification and [05.md](05.md), every Ark payment returned by `list_transactions` MUST use the common record envelope and the Ark `details` object defined above for `lookup_payment`.

## Notifications

When a wallet supports both this specification and [02.md](02.md), it SHOULD send `payment_received` when an incoming Ark payment becomes `settled`, and `payment_sent` when an outgoing Ark payment becomes `settled`.

For an Ark payment, the `notification` object is the common payment record envelope with `payment_type: "ark"`, the same object `lookup_payment` would return. It has no `invoice`, `payment_hash` or `preimage`.

Ark payment records, whether returned by `pay_ark_address`, `lookup_payment` or `list_transactions` or sent in a notification, never contain a `payment_hash`. Clients MUST NOT assume a `payment_hash` when `payment_type` is `ark`.

Clients SHOULD check `payment_type` before reading BOLT11 fields from a notification.

Ark payments to an address arrive without a prior request from the client, so notifications are the main way for a client to learn about them.

## Security and privacy considerations

- Ark addresses are reusable. A published address links every payment made to it. Wallets SHOULD return a fresh address from `make_ark_address`.
- The Ark server sees the sender and recipient VTXOs of every off-chain payment it cosigns.
- An Ark payment cannot be refused or deduplicated by the recipient. Clients that retry MUST use `idempotency_key`.
- Spending limits, such as the connection budgets in [13.md](13.md), apply to Ark payments as they do to Lightning payments. Both `amount` and `fees_paid` count toward the budget.
- Before showing a payment as received, a client SHOULD wait for the `settled` state, not `pending`.

## Example flow

1. The **client** calls `get_info`. The result lists `pay_ark_address` in `methods` and `XX` in `extensions`, and contains `"ark": {"server_pubkey": "038202bebd..."}`.
2. The recipient's profile, or a BIP-321 URI they shared, contains an Ark address. Examples of the two BOAT-0001 versions in use today:
    - version 0, used by Arkade: `ark1qzpq904am6clw3pgqwyh4p02708fy4xs0hcpwt7rwfdttuxsjameetl3ujgrw8089sl27rtp79aqcl0xspkahwnm4teg5lmhe47pxulw9m6rn8`
    - version 1, used by bark: `ark1pwh9vsmezqqpjy9akejayl2vvcse6he97rn40g84xrlvrlnhayuuyefrp9nse2yspqqjl5wpy`
3. The **client** checks whether the address is for the wallet's Ark server, as described under [`get_info`](#get_info). If it is not, the client pays a Lightning invoice instead.
4. The **client** calls `pay_ark_address` with the address, the amount and a fresh `idempotency_key`.
5. The **wallet service** checks the address, pays and returns a `transaction_id` and a `state`. If the response is lost, the **client** retries with the same `idempotency_key`, or calls `lookup_payment` with it. The payment is never made twice.
6. The recipient's **wallet service** sends a `payment_received` notification with `"payment_type": "ark"` once the payment is `settled`.

## Relationship to other specs

- [09.md](09.md) defines the generic payment lookup envelope used by the `ark` payment type.
- [05.md](05.md) defines `list_transactions`. Ark items use the same record shape as `lookup_payment`.
- [02.md](02.md) defines payment notifications. Clients use `lookup_payment` to reconcile the current Ark payment record.
- [13.md](13.md) defines connection budgets, which apply to `pay_ark_address`.
- [321.md](321.md) defines BIP-321 `pay` and `receive` for Lightning instructions. A BIP-321 URI can also carry an Ark address in an `ark` parameter. A client that finds one MAY pay it with `pay_ark_address`, and MAY fall back to a Lightning instruction from the same URI when it gets `UNSUPPORTED_ARK_ADDRESS`.
