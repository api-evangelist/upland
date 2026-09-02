---
name: upland-rumble-tournament
description: Create, fill, score and settle an Upland Rumble tournament, including the UPX entry fee and prize distribution rules.
api: Upland Developers API
base_url: https://api.prod.upland.me/developers-api
operations:
  - RumbleTournamentSettingsController_createRumbleTournamentSettings
  - RumbleTournamentSettingsController_findRumbleTournamentSettings
  - RumbleTournamentSettingsController_getRumbleTournamentSettings
  - RumbleTournamentSettingsController_editRumbleTournamentSettings
  - RumbleTournamentsController_create
  - RumbleTournamentsController_join
  - RumbleTournamentsController_closeRumbleTournamentRegistration
  - RumbleTournamentsController_startTournament
  - RumbleTournamentsController_registerScores
  - RumbleTournamentsController_getRumbleTournamentScoreboard
  - RumbleTournamentsController_getRumbleTournamentCurrentPrizeDistribution
  - RumbleTournamentsController_findRumbleTournamentParticipants
  - RumbleTournamentsController_getRumbleTournamentParticipantInfo
  - RumbleTournamentsController_findRumbleTournament
  - RumbleTournamentsController_resolveTournament
  - RumbleTournamentsController_cancelTournament
generated: '2026-09-02'
method: generated
source: >-
  openapi/upland-developers-api-openapi.json and
  https://docs.developers.upland.me/upland-developers/api-definitions/tournament-apis
---

# Run an Upland Rumble tournament

Tournaments move real UPX. Players pay to enter, your application scores them, and Upland
distributes the pot according to rules you defined up front.

## Steps

1. **Define settings once.** `POST /rumble-tournament-settings`
   (`RumbleTournamentSettingsController_createRumbleTournamentSettings`). This carries:
   - `prizeDistributionRules` — how the pot is split.
   - `registrationFeeInUpx` — what each player pays to join.
   - `developerFeePercentage` — your cut. **Range 0–10%.**
   - `stakeholderFeePercentage` — an optional third-party cut, e.g. a licensed brand account.
     **Range 0–10%.**
   Editing settings later (`RumbleTournamentSettingsController_editRumbleTournamentSettings`)
   affects **new tournaments only**. Existing tournaments keep a snapshot of the settings they were
   created with.
2. **Create the tournament.** `POST /rumble-tournaments` (`RumbleTournamentsController_create`) with
   the settings id, optionally overriding `registrationFeeInUpx` and naming a `stakeholderEosId`.
   The tournament starts in `WAITING_FOR_PARTICIPANTS`.
3. **Register players.** `POST /rumble-tournaments/{id}/join` (`RumbleTournamentsController_join`).
   **This is the one tournament operation that needs the player Bearer token**, not Basic auth.
   Every other tournament call uses App ID + Secret Key. Watch for
   `RumbleTournamentNewParticipantAdded` and `RumbleTournamentParticipantStatusUpdated`
   (`WAITING_PAYMENT` → `PROCESSING_PAYMENT` → `ACTIVE`, or `REMOVED`).
4. **Close registration** (optional). `PATCH /rumble-tournaments/{id}/close-registration`.
5. **Start.** `PATCH /rumble-tournaments/{id}/start`
   (`RumbleTournamentsController_startTournament`). You cannot start or close a tournament before
   the minimum participant count in the settings is met.
6. **Score.** `POST /rumble-tournaments/{id}/scores`
   (`RumbleTournamentsController_registerScores`). Read back with
   `GET /rumble-tournaments/{id}/scoreboard` and
   `GET /rumble-tournaments/{id}/prize-distribution`.
7. **Settle.** `POST /rumble-tournaments/{id}/resolve`
   (`RumbleTournamentsController_resolveTournament`). Status walks
   `IN_PROGRESS` → `DISTRIBUTING_PRIZES` → `CLOSED`, announced on
   `RumbleTournamentStatusUpdated` and finally `RumbleTournamentClosed`.

## Cancelling

`POST /rumble-tournaments/{id}/cancel` (`RumbleTournamentsController_cancelTournament`) refunds all
UPX to the original owners and fires `RumbleTournamentCanceled`. **You cannot cancel while a
participant payment is in progress on the blockchain.** No deadline for cancellation is published —
only that blocking condition.

## Rules that will bite you

- **Two auth schemes in one flow.** Only `join` takes the Bearer token. Mixing them up returns 401,
  a response the contract never declares.
- **Failure is a status, not a status code.** `RumbleTournamentStatusUpdated` with status `FAILED`
  and an optional `error` string is how a tournament dies. HTTP will have said 2xx long before.
- **No idempotency.** Re-posting scores after a timeout has no documented deduplication.
- **Fees are capped at 10% each.** Values outside 0–10 are rejected.

## Related

- `errors/upland-problem-types.yml` (tournament and participant status vocabularies)
- `asyncapi/upland-webhooks.yml`
