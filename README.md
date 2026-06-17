# Top-up Wizard — Xahau Hooks

> **Credits / attribution.** This work is based on the original Topup example by
> **Satish** ([@Satish_nl](https://x.com/Satish_nl) on X), published at
> <https://github.com/technotip/HookExamples/tree/main/Topup>. All credit for the
> original concept and implementation goes to the author; this repository is a
> derivative and is **not** claimed as original work.

A pair of Xahau Hooks that implement **automatic XAH balance top-ups** between two
accounts:

- **`request.c`** runs on the account that *wants* to stay funded (the **user / spender**).
- **`payment.c`** runs on the account that *pays* the top-ups (the **treasury / funder**).

When the user spends XAH and their balance drops below a minimum they defined, the
`request.c` hook automatically asks the treasury for money. The treasury's `payment.c`
hook checks that the requester is whitelisted and below its own threshold, then emits a
`Payment` back to the user. The whole cycle happens on-ledger, with no off-chain bot.

---

## How the pair works

```
  USER ACCOUNT (request.c)                 TREASURY ACCOUNT (payment.c)
  ────────────────────────                 ────────────────────────────
  1. User sends a Payment in XAH
  2. Hook fires (HookOn = Payment)
  3. Projected balance = balance − amount
  4. If projected balance < A (minimum)
        │
        └── emits an Invoke ───────────────►  5. Hook fires (HookOn = Invoke)
                                              6. Sender is looked up in the whitelist
                                              7. If whitelisted, reads sender balance
                                              8. If balance < B (threshold)
        ◄────────── emits a Payment ──────────     emits Payment(amount) to the sender
  9. User receives the top-up
```

The two thresholds are independent and set by each account owner:

| Hook        | Threshold | Meaning                                                             |
|-------------|-----------|---------------------------------------------------------------------|
| `request.c` | `A`       | Minimum balance the **user** wants to keep. Below it, asks for help.|
| `payment.c` | `B`       | Balance below which the **treasury** agrees to top an account up.   |

> Set `B ≥ A` so the treasury actually agrees to pay when the user asks. The per-account
> top-up amount the treasury sends is configured separately in the whitelist (see below).

---

## Public HookHashes

The compiled WASM is already published on **both** Xahau Mainnet and Testnet under the same
hash, so you do **not** need to compile or upload anything — you can install by reference.

| Hook        | Role             | HookHash (Mainnet & Testnet)                                       | Fires on |
|-------------|------------------|--------------------------------------------------------------------|----------|
| `payment.c` | Treasury / funder| `6B2E2ABF0AAB0899165EB94F65459DE4A8DFB2AA22AF2484758EFCE8FD8F5D9D` | `Invoke` |
| `request.c` | User / spender   | `210370D99EFA10B6AD9924E33D56B00133045BDD2C0A3AC981E632E112F4CEFC` | `Payment`|

`HookOn` values (canonical):

- `payment.c` → `FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF7FFFFFFFFFFFFFFFFFFBFFFFF` (Invoke only)
- `request.c` → `FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFBFFFFE` (Payment only)

---

## `payment.c` — the treasury hook

Install this on the account that holds the funds and pays the top-ups.

### Hook parameters (set at `SetHook` time)

| Name | Bytes | Type | Required | Description |
|------|-------|------|----------|-------------|
| `B`  | 8     | XFL  | yes      | Balance threshold. A whitelisted account is only topped up while its balance is **below** `B`. |

If `B` is missing the hook rolls back with
`"Topup: Misconfigured. Balance of the requesting account, while requesting."`

### Whitelist (Hook State)

The treasury keeps a whitelist in Hook State. Each entry is keyed by a 20-byte account ID
and stores that account's top-up **amount** (8-byte XFL).

**To add or update an entry**, the treasury owner sends an `Invoke` *to its own account*
with these **transaction parameters**:

| Name | Bytes | Type      | Description |
|------|-------|-----------|-------------|
| `D`  | 20    | AccountID | Account to add to / update in the whitelist. |
| `A`  | 8     | XFL       | Top-up amount sent to that account. |

The account must exist on-ledger (it is validated via a keylet) or the hook rolls back.

**To remove an entry**, send the same `Invoke` with parameter `D` only (omit `A`). The
state entry is deleted (`"Topup: Removed from whitelist."`).

### Top-up request handling

When **any other account** sends an `Invoke` to the treasury:

1. The sender is looked up in the whitelist. If not found → `"Topup: Unauthorized topup request."`
2. The sender's current balance is read from the ledger.
3. If `balance < B`, a `Payment` of the whitelisted amount is emitted to the sender
   (`"Topup: Topup fulfilled successfully."`).
4. Otherwise → `"Topup: Not yet eligible to request topup."`

### Result messages

`Configured Successfully` · `Removed from whitelist` · `Topup fulfilled successfully` ·
`Not yet eligible to request topup` · `Unauthorized topup request` · `Failed To Emit` ·
various `Misconfigured` / `Fetching Keylet Failed` errors.

---

## `request.c` — the user hook

Install this on the account that wants to be topped up automatically.

### Hook parameters (set at `SetHook` time)

| Name | Bytes | Type      | Required | Description |
|------|-------|-----------|----------|-------------|
| `A`  | 8     | XFL       | yes      | Minimum balance to maintain. |
| `D`  | 20    | AccountID | yes      | Treasury account (the one running `payment.c`) to send the request to. |

If either is missing the hook rolls back with a `"Misconfigured"` message.

### Behaviour

The hook fires on the account's **own outgoing `Payment`** transactions:

1. Reads the payment's `Amount`. If it is **not** native XAH (i.e. an issued-currency
   amount, not 8 bytes), the hook does nothing and accepts
   (`"Topup: Non XAH Transaction Successful."`).
2. Computes the projected balance: `current balance − amount sent`.
3. If the projected balance is **below `A`**, it emits an `Invoke` to the treasury account
   `D` (`"Topup: Payment Request Sent Successfully."`). That `Invoke` triggers `payment.c`
   on the treasury, which decides whether to pay.
4. Otherwise it simply accepts (`"Topup: Payment Successful."`).

> The hook only watches XAH spending. Issued-currency (trustline) payments are ignored.

---

## Installation

Both hooks ship with their `cbak` export (required by Xahau, otherwise `SetHook` is rejected
with `temMALFORMED`). Because the definitions already exist on both networks, install **by
hash reference** — no WASM upload, no `CreateCode`.

### Install by HookHash (recommended)

Submit a `SetHook` transaction from each account referencing the published hash. Conceptual
shape of the `Hook` object:

**Treasury account → `payment.c`**

```jsonc
{
  "Hook": {
    "HookHash": "6B2E2ABF0AAB0899165EB94F65459DE4A8DFB2AA22AF2484758EFCE8FD8F5D9D",
    "HookOn":   "FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF7FFFFFFFFFFFFFFFFFFBFFFFF",
    "HookNamespace": "<your 32-byte namespace>",
    "HookParameters": [
      { "HookParameter": { "HookParameterName": "42",                 // "B"
                           "HookParameterValue": "<8-byte XFL threshold, little-endian hex>" } }
    ]
  }
}
```

**User account → `request.c`**

```jsonc
{
  "Hook": {
    "HookHash": "210370D99EFA10B6AD9924E33D56B00133045BDD2C0A3AC981E632E112F4CEFC",
    "HookOn":   "FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFBFFFFE",
    "HookNamespace": "<your 32-byte namespace>",
    "HookParameters": [
      { "HookParameter": { "HookParameterName": "41",                 // "A"
                           "HookParameterValue": "<8-byte XFL minimum, little-endian hex>" } },
      { "HookParameter": { "HookParameterName": "44",                 // "D"
                           "HookParameterValue": "<20-byte treasury AccountID hex>" } }
    ]
  }
}
```

> `HookParameterName` is the hex of the ASCII letter: `A` = `41`, `B` = `42`, `D` = `44`.

### Build from source (optional)

If you prefer to compile and verify the WASM yourself:

1. Use the Xahau Hooks toolchain (`hookapi.h`, `wasmcc`/`clang` + `guard_checker`/`wasm-opt`),
   e.g. the [hooks-builder](https://hooks-builder.xrpl.org) or the `xrpl-hooks` Docker image.
2. Compile each `.c` to `.wasm` (the prebuilt `payment.wasm` / `request.wasm` are included).
3. Submit `SetHook` with `CreateCode` (the WASM hex) instead of `HookHash`. The resulting
   hash must match the values above for a byte-identical build.

---

## Encoding the parameters

- **Account IDs (`D`)** are the 20-byte raw AccountID, *not* the `r...` base58 address.
  Decode the classic address to its 160-bit AccountID and use the hex.
- **Amounts / thresholds (`A`, `B`)** are **XFL** (Xahau's 64-bit float format), supplied as
  an 8-byte value. They are *not* plain drops. The hook reads the parameter directly as a
  64-bit value, so provide the XFL in **little-endian** byte order.
  - Example: 5 XAH = 5,000,000 drops → build the XFL for `5,000,000 × 10⁻⁶` and store its
    8 little-endian bytes. Use an XFL helper (e.g. `floatToXfl` from the hooks JS utilities)
    rather than hand-encoding.

> **Helper tools.** To convert addresses, amounts and text into the hex blobs these hooks
> expect, you can use:
> - XRPL Hex Visualizer — <https://transia-rnd.github.io/xrpl-hex-visualizer/>
> - hooks.services tools — <https://hooks.services/tools>

---

## Configuration walkthrough

1. **Treasury**: `SetHook` `payment.c` with parameter `B` (e.g. "top up anyone below 20 XAH").
2. **Treasury**: send an `Invoke` to itself with tx params `D` = the user's AccountID and
   `A` = the top-up amount (e.g. 10 XAH). This whitelists the user.
3. **User**: `SetHook` `request.c` with `A` = minimum balance (e.g. 5 XAH) and `D` = the
   treasury AccountID.
4. **User**: spend XAH normally. When a payment would drop the balance below 5 XAH, the user
   hook fires an `Invoke`, the treasury hook checks the whitelist and balance, and sends the
   10 XAH top-up automatically.

To stop topping up an account, the treasury sends an `Invoke` to itself with `D` only.

---

## Testing on Testnet

Always validate on **Xahau Testnet** before mainnet:

- Faucet & explorer: <https://xahau-test.net>
- The same HookHashes are installable on Testnet, so the install steps are identical — only
  the network endpoint and funded test accounts change.
- Verify the on-chain definition before relying on it (HookOn, fee, reference count) using
  any Xahau tooling or `account_objects` / `hook` ledger queries.

---

## Notes & limitations

- Only **native XAH** is handled. Issued-currency payments are ignored by `request.c`.
- Each emitted transaction (`Invoke` from the user, `Payment` from the treasury) pays its own
  fee from the emitting account; keep both accounts funded above the reserve + fee.
- The treasury only pays accounts present in its whitelist state — there is no open faucet.
- `request.c` triggers on the user's outgoing payments; it does not poll. If the user never
  transacts, no top-up is requested even if the balance is low.
- Both hooks export a trivial `cbak` (no reaction to emitted-transaction outcomes).
