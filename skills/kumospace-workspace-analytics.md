---
name: kumospace-workspace-analytics
description: Pull Kumospace workspace attendance and presence-status analytics, paging through results with the cursor envelope.
api: kumospace:api
generated: '2026-07-19'
method: generated
source: openapi/kumospace-api-openapi.json
base_url: https://api.kumospace.com
operations:
  - GetAnalytics
  - GetStatusAnalytics
---

# Pull Kumospace workspace analytics

Retrieve who spent time where, and presence-status history.

## Before you start

- Both operations require `Authorization: Bearer <firebase-id-token>` and are host/admin-scoped.
- Both are paginated with the standard cursor envelope. There is no total count — only
  `metadata.count` (items in this page) and `metadata.next` (cursor, absent when exhausted).
- This data is per-person time tracking: rows carry `userEmail`, `userDisplayName` and `userId`. Handle it
  as employee monitoring data and apply the same care you would to an HR export.

## Steps

1. **Attendance.** `GetAnalytics` (`GET /v1/analytics/attendance`) returns
   `PaginatedResponse_HostAnalyticsRoomStat_`. Each `HostAnalyticsRoomStat` carries `duration`,
   `userId`, `userEmail`, `userDisplayName`, `roomId` and `roomDisplayName` — time spent per user per room.
   Scope the query with `space`, `start` and `end` where supported.

2. **Presence status.** `GetStatusAnalytics` (`GET /v1/analytics/status`) returns
   `PaginatedResponse_AnalyticsEvent_` — presence-status events over time, which is what the Zoom and
   Microsoft Teams status integrations feed.

3. **Page through.** For either call:
   - send `limit` on the first request;
   - read `metadata.next` from the response;
   - re-send the same query with `next` set to that cursor;
   - stop when `metadata.next` is absent.

## Notes and gotchas

- Durations are `number` with `format: double`. The spec does not state the unit; validate against a
  known session before building reporting on top of it.
- `roomId` joins to `ApiRoom.id`, and rooms arrive inline on `ApiSpace.rooms` from `GetSpaceByName` — one
  call gives you the id-to-name map for enrichment.
- There is no export endpoint and no scheduled delivery. Recurring reporting means walking the cursor on a
  schedule and storing results yourself.
- The API declares no rate limits and no 429 response, but that is an absence of documentation rather than
  an absence of limits. Page politely and back off on any non-2xx.
