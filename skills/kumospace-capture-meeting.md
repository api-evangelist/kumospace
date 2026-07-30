---
name: kumospace-capture-meeting
description: Record a Kumospace session and retrieve the artifacts — start and stop a recording, share it, then pull the transcript, VTT captions and AI-generated summary.
api: kumospace:api
generated: '2026-07-19'
method: generated
source: openapi/kumospace-api-openapi.json
base_url: https://api.kumospace.com
operations:
  - CreateRecording
  - HandleRecordingStop
  - GetRecordingById
  - GetRecordingsByUser
  - GetRecordingAccessLink
  - ShareRecording
  - GetTranscription
  - GetSummary
  - GetVtt
---

# Capture and retrieve a Kumospace meeting

Record a session, then collect the transcript, captions and summary.

## Before you start

- All operations require `Authorization: Bearer <firebase-id-token>`.
- Two identifiers matter and they are **not** the same thing:
  - `recordingId` — a single `ApiRecording`.
  - `recordingGroupId` — the grouping that a transcript hangs off. `ApiTranscription` references
    `recordingGroupId`, not `recordingId`. Read the group id off the recording before asking for a transcript.
- Recording and transcription produce personal data. `ApiRecording.visibility` (`RecordingVisibility`) and
  `sharedWithFirebaseUids` govern who can see a recording — set them deliberately rather than relying on a
  default.
- There is **no idempotency key** and **no declared error response**. Poll rather than assume.

## Steps

1. **Start recording.** `CreateRecording` (`POST /v1/recordings`). The response is an `ApiRecording`
   carrying `id`, `status`, `startTimestamp`, `roomId`, `spaceName` and `recordingGroupId`.

2. **Stop it.** `HandleRecordingStop` (`POST /v1/recordings/{recordingId}/stop`).

3. **Poll until processing completes.** `GetRecordingById` (`GET /v1/recordings/{recordingId}`) and read
   `status` and `duration`. The spec does not enumerate the status values or publish a processing SLA, so
   back off between polls rather than tight-looping.

4. **Get a playable link.** `GetRecordingAccessLink`
   (`GET /v1/recordings/{recordingId}/accessLink`). Treat the returned link as a short-lived credential —
   fetch it on demand instead of persisting it.

5. **Share it.** `ShareRecording` (`PUT /v1/recordings/{recordingId}/share`) adjusts `visibility` and
   `sharedWithFirebaseUids`.

6. **Pull the transcript.** `GetTranscription` (`GET /v1/transcription`) — a list operation, so it uses
   the standard cursor pagination: pass `limit`, read `metadata.next`, pass it back as `next`. Match on
   `recordingGroupId` to find the transcript for your session, and take its `id`.

7. **Pull the derived artifacts**, using the transcription `id` from the previous step:
   - `GetSummary` (`GET /v1/transcription/{id}/summary`) — the AI-generated meeting summary. Note that
     `ApiTranscription.summaryJob` implies the summary is produced asynchronously, so it may not be ready
     the moment the transcript exists.
   - `GetVtt` (`GET /v1/transcription/{id}/vtt`) — WebVTT captions, for players and downstream indexing.

## Notes and gotchas

- To find prior recordings for a workspace rather than a single one, use `GetRecordingsByUser`
  (`GET /v1/recordings/spaces/{spaceName}`).
- `HandleTranscriptionStarted` (`POST /v1/transcription/started`) and `HandleTranscriptionStopped`
  (`POST /v1/transcription/stopped`) exist but read as internal lifecycle callbacks fired by the client
  or the media pipeline. Do not drive them by hand as part of an integration.
- Kumospace offers **no outbound webhooks**, so there is no completion event to subscribe to for either
  recording processing or summarization. Polling is the only available pattern.
- Recordings in a HIPAA-enabled space inherit that space's compliance posture; check `hipaaEnabled` on
  `ApiSpace` and `IsUserHipaaCompliant` (`GET /v1/users/{userId}/hipaaCompliant`) before moving recording
  content or transcripts into any external system.
