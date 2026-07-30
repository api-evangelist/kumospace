---
name: kumospace-provision-space
description: Create and lay out a Kumospace virtual office — provision a space, add rooms, place furniture, and set the default landing spot for members.
api: kumospace:api
generated: '2026-07-19'
method: generated
source: openapi/kumospace-api-openapi.json
base_url: https://api.kumospace.com
operations:
  - CreateSpace
  - GetSpacesByOwner
  - GetSpaceByName
  - UpdateSpace
  - CreateFurniture
  - UpdateHome
---

# Provision a Kumospace space

Stand up a new virtual office and give it a usable layout.

## Before you start

- Every call needs `Authorization: Bearer <firebase-id-token>`. The token is issued by Firebase
  Authentication; the API declares a single `firebase` http/bearer scheme and applies it to this whole flow.
- A space is addressed by its `name` — a url-safe slug, not a numeric id. `name` is the primary key in
  every downstream path, so choose it deliberately; `displayName` is the human label and can change freely.
- The API declares **no error responses at all**. Treat any non-2xx as opaque: log the status and body,
  and do not assume a machine-readable error shape exists.
- There is **no idempotency key**. `CreateSpace` and `CreateFurniture` are not safe to blindly retry — a
  retry after an ambiguous timeout may create a duplicate. Re-read with `GetSpacesByOwner` before retrying.

## Steps

1. **Check what already exists.** `GetSpacesByOwner` (`GET /v1/spaces`) returns the spaces the
   authenticated user owns. Do this first so a retry does not create a second space.

2. **Create the space.** `CreateSpace` (`POST /v1/spaces`). Supply the `name` slug and `displayName`.
   The response is an `ApiSpace`, which carries `uid`, `ownerId`, `tier`, `memberLimit`, `permissions`
   and an inline `rooms` array.

3. **Read the space back.** `GetSpaceByName` (`GET /v1/spaces/{name}`). Because `ApiSpace` inlines its
   full `rooms` array, this single call gives you every room id you need for the layout steps — there is
   no separate list-rooms operation.

4. **Adjust space-level settings.** `UpdateSpace` (`PUT /v1/spaces/{name}`) for things like
   `description`, `emailDomains` (which control auto-join by email domain), `chatSettings`,
   `hideInviteButton` and `requireDesktopApp`.

5. **Place furniture in a room.** `CreateFurniture`
   (`POST /v1/spaces/{name}/rooms/{id}/furniture`) — note this path names the room parameter `id`, while
   the update and delete paths name it `roomId`. Furniture carries a behavior from the
   `FurnitureBehaviorType` enum: `whiteboard`, `youtube`, `spotify`, `soundCloud`, `pictureFrame`,
   `stickyNote`, `timer`, `link`, `kiosk`, `broadcast`, `elevator`, games such as `chess`, `crossword`
   and `jstris`, and others. Pull the stock catalog first from `GetStockFurniture`
   (`GET /v1/stockFurniture`) — that endpoint needs no authentication.

6. **Set the landing spot.** `UpdateHome` (`PUT /v1/spaces/{name}/home`) defines where members spawn when
   they enter. `DeleteHome` (`DELETE /v1/spaces/{name}/home`) clears it.

## Notes and gotchas

- `CreateRoom` is **ambiguous in the published spec**: the same `operationId` is assigned to both
  `POST /v1/spaces/{name}/rooms` (creating a room) and `POST /v1/spaces/{name}/rooms/{roomId}/zones`
  (creating a zone). Resolve by path and method, not by `operationId`, and expect generated clients to
  collide on this name.
- Room layouts can be seeded from `GetRoomTemplates` (`GET /v1/roomTemplates`, unauthenticated) and zone
  layouts from `GetZoneTemplates` (`GET /v1/zoneTemplates`, unauthenticated).
- Trial state lives on the space: `GetFreeTrialInfo` (`GET /v1/spaces/{name}/trial`) and
  `StartInternalFreeTrial` (`PUT /v1/spaces/{name}/trial`).
- HIPAA mode is a space-level flag (`hipaaEnabled` on `ApiSpace`), available on paid plans.
