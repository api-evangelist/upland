---
name: upland-escrow-container
description: Run the full Upland escrow container lifecycle — create, take player assets in, resolve or refund — with the asynchronous settlement and irreversibility rules that govern it.
api: Upland Developers API
base_url: https://api.prod.upland.me/developers-api
operations:
  - EscrowController_create
  - EscrowController_getContainer
  - UserController_putAssetsInEscrowContainer
  - EscrowController_putAssetsInEscrowContainerWithPermissionDelegation
  - EscrowController_lockContainer
  - EscrowController_unlockContainer
  - EscrowController_refresh
  - EscrowController_removeTransaction
  - EscrowController_resolve
  - EscrowController_refund
generated: '2026-09-02'
method: generated
source: >-
  openapi/upland-developers-api-openapi.json and
  https://docs.developers.upland.me/upland-developers/api-definitions/escrow-container-management
---

# Run an Upland escrow container

An escrow container is a vault Upland controls on your application's behalf. Players move real
assets into it, your application decides who gets what, and resolution is **final and
irreversible**. Everything below is written around that one fact.

## The settlement model

Every asset-moving call returns 2xx to say Upland accepted the instruction. Nothing has moved yet.
The outcome arrives on your webhook, and Upland's own docs warn that even a *signed* transaction is
not yet complete on-chain. Blockchain ownership change is documented as taking up to three minutes.

**Never treat an HTTP 2xx on these operations as settlement.** Wait for the terminal event.

## Steps

1. **Create the container.** `POST /containers` (`EscrowController_create`). Its expiration is
   governed by the "Expiration Time in Hours" set on your application. You can also cap how many
   players may join.
2. **Bring assets in.** Two paths, and they are not interchangeable:
   - `POST /user/join` (`UserController_putAssetsInEscrowContainer`) — **player Bearer token**. The
     player must sign.
   - `POST /containers/{containerId}/join`
     (`EscrowController_putAssetsInEscrowContainerWithPermissionDelegation`) — **Basic auth only**,
     available if your account is approved for the alpha Permission Delegation feature. Moves assets
     from the *developer's own* account, takes no EOS ID.
   Players must be at one of your Dev Shop addresses to place assets into escrow in production.
3. **Track each transaction.** Watch the webhook stream:
   `TransactionToEscrowCreated` → `TransactionToEscrowSigned` → `TransactionToEscrowFinal`.
   Only `TransactionToEscrowFinal` means the asset is yours to allocate. The failure branches are
   `TransactionToEscrowFailure`, `TransactionToEscrowExpired` (**player did not sign within ten
   minutes**) and `TransactionToEscrowRejected`.
4. **Inspect.** `GET /containers/{containerId}` (`EscrowController_getContainer`) returns the
   container with its assets and each transfer's status:
   `user_signature_requested`, `changing_ownership`, `in_escrow`, `expired`, `rejected`, `refunded`,
   `removed`.
5. **Freeze if you need to.** `POST /containers/{containerId}/lock` and `/unlock`
   (`EscrowController_lockContainer` / `EscrowController_unlockContainer`).
6. **Buy time.** `POST /containers/{containerId}/refresh-expiration-time`
   (`EscrowController_refresh`) pushes the expiry out. If you let it lapse, Upland fires
   `ContainerExpired` and returns every asset to its original owner without charge.
7. **Resolve.** `POST /containers/{containerId}/resolve` (`EscrowController_resolve`). **Every asset
   in the container must be `in_escrow` first.** Watch for `TransactionFromEscrowCreated` then
   `TransactionFromEscrowFinal`; on failure, `TransactionFromEscrowFailure`.

## Reversal, and where it stops

| Situation | What you can do | Window |
|---|---|---|
| Transaction sent, player has not acted | `DELETE /containers/{containerId}/transactions/{transactionId}` (`EscrowController_removeTransaction`) | Before the player accepts or rejects |
| Player never signs | Nothing — it expires by itself | 10 minutes |
| Assets are in escrow, container unresolved | `POST /containers/{containerId}/refund` (`EscrowController_refund`) | **Not published — do not assume one** |
| Container left alone | Automatic return to owners on `ContainerExpired` | Your application's configured expiration hours |
| Container resolved | **Nothing.** Final and irreversible. | — |

## Rules that will bite you

- **No idempotency key exists.** Retrying a timed-out `POST /containers` can create a second
  container. Record your own request identity before you send.
- **No rate limits are published and no rate-limit headers are returned.** Pace yourself against the
  three-minute settlement time, not against a limit you cannot see.
- **Asset eligibility changes.** Block Explorers, Structured Ornaments, Spirit Legits, UPX and 3D
  assets are usable; branded assets such as FIFA and Stock Car need permission. Categories have been
  withdrawn and reinstated across releases — check the release notes before relying on one.
- **3D assets land in Impound Grounds.** A player who wins a 3D asset collects it from their wallet;
  it is unusable until collected.

## Related

- `conventions/upland-conventions.yml` (reversibility block)
- `asyncapi/upland-webhooks.yml`
- `errors/upland-problem-types.yml`
