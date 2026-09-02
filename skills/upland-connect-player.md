---
name: upland-connect-player
description: Connect an Upland player to a third-party application and obtain the player Bearer token used for all player-scoped calls.
api: Upland Developers API
base_url: https://api.prod.upland.me/developers-api
operations:
  - AuthController_otpInit
  - UserController_getUserProfile
  - UserController_getBalances
generated: '2026-09-02'
method: generated
source: >-
  openapi/upland-developers-api-openapi.json and
  https://docs.developers.upland.me/upland-developers/api-definitions/upland-users-authentication
---

# Connect an Upland player

Upland has no OAuth redirect. A player authorizes an application by typing a short code into their
own Upland account, and the resulting token arrives at the application's webhook — never in an HTTP
response. Plan for that before you start: **this flow cannot complete without a reachable webhook URL
registered on the application.**

## Before you start

- The application must exist in the Developers Portal with a Webhook URL and Webhook Access Token set.
- You hold the App ID and App Secret Key. They are the username and password of an HTTP Basic credential.
- Sandbox and production are separate universes. Sandbox credentials work only against
  `https://api.sandbox.upland.me/developers-api`.

## Steps

1. **Request a connection code.** `POST /auth/otp/init` (`AuthController_otpInit`) with HTTP Basic
   auth. A 404 here means "Application not found" — check that the App ID in the credential matches
   an active application.
2. **Give the code to the player.** Display, text or email it. Upland's docs put no stated lifetime
   on it, only that it can expire.
3. **Wait for the webhook.** The player pastes the code into their Upland account. Upland then POSTs
   to your webhook URL:
   - `AuthenticationSuccess` — `{code, userId, accessToken}`. Store `accessToken`; it is the Bearer
     token for every player-scoped call.
   - `AuthenticationFailure` — `{code, message}`. The code expired or the flow failed. Start over at
     step 1; do not retry the same code.
4. **Verify the token.** `GET /user/profile` (`UserController_getUserProfile`) with
   `Authorization: Bearer <accessToken>`. It returns the player's profile including their home
   address and their `eosId`, the on-chain account name.
5. **Read what you need.** `GET /user/balances` (`UserController_getBalances`) for UPX and related
   balances.

## Rules that will bite you

- **There is no 401 in the contract.** Every operation requires credentials and not one declares a
  401 response. Handle it anyway; the live API returns
  `{"statusCode":401,"message":"Unauthorized"}`.
- **Handle disconnection.** When the player disconnects your app, Upland POSTs
  `UserDisconnectedApplication` with `{appId, userId}`. Discard the token on that event; nothing
  else tells you it is dead.
- **No token lifetime is published.** Treat a 401 on a previously working token as a re-connect
  trigger, not an outage.
- **Rotating your own secret is destructive.** The App Secret Key cannot be retrieved. Rotation
  means inactivating and reactivating the application, which issues a new token and breaks live
  integrations.
- **Authenticate the inbound webhook yourself.** Upland presents the Webhook Access Token you chose.
  There is no HMAC signature and no replay protection.

## Related

- `authentication/upland-authentication.yml`
- `asyncapi/upland-webhooks.yml`
