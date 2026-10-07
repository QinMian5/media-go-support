# Media Go receiver contract

Public receiver protocol for Media Go 0.1.0.

## 1. Capture and delivery

A media item uses an immutable snapshot of its capture button: purpose, delivery mode, segment length, and annotation fields. Later edits or deletion of the button do not change existing items.

Audio is PCM16 WAV, 48 kHz, mono. Whole recording sends one complete WAV using `file` only after the user stops and confirms annotations; nothing for that item is sent before confirmation. There is no application-level recording duration limit. Continuous upload closes and uploads segments while recording, with a configured length of 1–600 whole seconds. A nonempty final segment may be shorter. After annotation confirmation, it submits a manifest and capture outcome using `finalize`.

Annotations can be edited before confirmation. The client validates required fields and types against the button snapshot. Confirmation fixes the final request; retries retain its content. Whole recording allows Record again and Discard before confirmation. Record again creates a new media ID, preserving the button snapshot and annotation draft. Continuous upload does not offer these actions because segments may already have been sent.

Whole recording retries the complete file. Continuous upload retries segments rather than resuming byte ranges. A receiver can acknowledge completed delivery once the audio segments, manifest, final annotations, and capture outcome have been durably saved and checked; merging audio or further processing need not finish first.

## 2. One URL, four operations

POST every operation to the configured receiver URL using `multipart/form-data`.

| `metadata.operation` | Meaning | Parts |
| --- | --- | --- |
| `file` | Complete WAV from Whole recording | One `file`, one `metadata` |
| `segment` | One Continuous upload segment | One `file`, one `metadata` |
| `finalize` | Manifest, capture outcome, final annotations | One `metadata` |
| `test` | Connection test without persistence | One `metadata` |

The `metadata` part is a serialized UTF-8 JSON object with Content-Type `application/json`. The `file` part has a filename and Content-Type `audio/wav`. Validate its format and nonzero valid audio frames. Filenames are not media identities.

If configured, send the complete `Authorization` header value; otherwise omit it. Credentials are not placed in metadata, audio, or plaintext queue content. Use standard multipart requests, not a persistent streaming connection. Filenames use UTF-8 percent encoding in the `filename` parameter; decode accordingly. See [RFC 7578](https://datatracker.ietf.org/doc/html/rfc7578#section-4).

## 3. Identity and annotations

Every operation except `test` includes:

| Field | Meaning |
| --- | --- |
| `operation` | `file`, `segment`, or `finalize` |
| `media_id` | Client-generated, persisted UUID shared by all operations for the recording |
| `purpose` | Arbitrary button-snapshot text; no application-level length limit; send `""` when empty |
| `annotations` | JSON object derived from annotation fields; `{}` when empty |

New captures get new IDs; retries keep their IDs. Purpose and delivery mode are fixed for a media ID. Do not mix `file` with `segment`/`finalize` under the same ID.

Users define only the contents of `annotations`, not protocol field names or structure. Dotted keys form nested objects: `speaker.name` produces `{"speaker":{"name":"Alex"}}`. Each level contains one or more letters, digits, or underscores. Keys cannot duplicate or conflict as parent and child, such as `speaker` and `speaker.name`.

Text and single-choice values are JSON strings, numbers are JSON numbers, and toggles always send JSON booleans. Whitespace-only text is unset. Omit unset optional fields other than toggles, and omit empty parent objects. Numbers can be negative or fractional with no application-level range restriction. Field names, order, defaults, and required-value validation belong to the client; a receiver does not need button definitions.

`segment` always sends `annotations: {}`. Final annotations arrive with `finalize`, or with `file` for Whole recording. Retried segments must not overwrite final annotations. Save annotation objects without assigning special meaning to a particular key.

## 4. Whole recording

Example metadata, with a sample UUID:

```json
{
  "operation": "file",
  "media_id": "00000000-0000-4000-8000-000000000001",
  "purpose": "",
  "annotations": {
    "kind": "recording",
    "speaker": {"name": "Alex"},
    "offset": -1.5,
    "consent": false
  }
}
```

Include its nonempty WAV in the `file` part. Receive the entire request, validate metadata and WAV, and durably save both before returning a save receipt. File hashes are not required.

## 5. Segments and timeline

```json
{
  "operation": "segment",
  "media_id": "00000000-0000-4000-8000-000000000002",
  "purpose": "Meeting notes",
  "annotations": {},
  "segment": {"index": 0, "start_frame": 0, "frame_count": 28800000}
}
```

`index` starts at zero, increases, and is never reused. The client prioritizes the earliest unacknowledged segment. Receivers identify order using indices and the audio timeline, not HTTP arrival order.

`start_frame` is a nonnegative integer; `frame_count` is a positive integer. Both use the 48 kHz timeline. The protocol limit is 600 seconds, or 28,800,000 frames. The client also caps each segment at its button-snapshot length multiplied by 48,000. The nonempty final segment may be shorter. Booleans are not integer frame counts.

The configured segment length is a client splitting rule and is not an additional request field. Receivers check the protocol limit and manifest. Read WAV structure to verify actual valid frames; total file bytes are not frame counts. Further processing can begin from persisted continuous segments without waiting for `finalize`; processing results are outside this protocol.

## 6. Finalization and manifest

This example is a complete 602-second recording using a 600-second segment length and a two-second final segment. `finalize` has no file part.

```json
{
  "operation": "finalize",
  "media_id": "00000000-0000-4000-8000-000000000002",
  "purpose": "Meeting notes",
  "annotations": {"meeting": {"topic": "Weekly review"}},
  "capture": {"end_reason": "stopped", "completeness": "complete"},
  "segments": [
    {"index": 0, "start_frame": 0, "frame_count": 28800000},
    {"index": 1, "start_frame": 28800000, "frame_count": 96000}
  ]
}
```

The manifest contains at least one valid segment, sorted by index without duplicates. Each entry must match a received segment's index and time range. Every declared segment must exist, and every received segment for that recording must appear in the manifest. Do not shrink the manifest to hide pending or lost segments.

A complete recording starts at frame zero and has contiguous, nonoverlapping ranges. Preserve known positions when audio is missing; do not remove gaps to pretend the timeline is complete. Save and verify the manifest, final annotations, and capture outcome before acknowledging `finalize`. Saved requests may still be retried, but content cannot be overwritten or new segments added. Segments can upload before annotation confirmation; `finalize` cannot.

## 7. Interrupted recordings

`capture.end_reason` is `stopped` for normal stopping or `interrupted` for an unexpected interruption. `capture.completeness` is:

- `complete`: no known or unresolved audio loss; the complete range can be verified.
- `incomplete`: known audio loss.
- `unknown`: whether or how much audio was lost is uncertain.

Readable recovered WAV files alone do not prove completeness. Use `unknown` when final audio preservation is uncertain and `incomplete` for known loss. Do not invent a missing duration.

Continuous upload retries recoverable segments before `finalize`, which still requires user annotation confirmation. Once the manifest and capture outcome are saved, the item can become Uploaded while retaining its interruption or completeness warning. No recoverable audio means finalization cannot succeed.

Recovered Whole recording audio still needs confirmation before `file`. It retains local warnings but adds no `capture` field to `file`. Capture loss and missing delivery files are different: `incomplete` or `unknown` cannot bypass upload of files declared in a manifest.

## 8. Save receipts and connection tests

After durable persistence, `file`, `segment`, and `finalize` return HTTP 200, Content-Type `application/json`, and:

```json
{"status":"received"}
```

The client requires a complete, valid JSON object whose `status` strictly equals the string `received`. Additional fields are not success conditions. `processing`, a missing status, invalid or truncated JSON, or HTTP success alone cannot mark an item Uploaded.

A `file` receipt acknowledges the entire WAV and final annotations. A `segment` receipt acknowledges only that segment and its metadata. A `finalize` receipt acknowledges the verified recording manifest, declared audio, final annotations, and capture outcome.

The client associates receipts with the fixed local request identity and receiver. A lost receipt leaves the result unknown and causes a retry. A client cannot independently verify a receiver's false persistence claim; the protocol relies on honest save acknowledgments.

A connection test sends only `metadata`, with current Authorization if configured:

```json
{"operation":"test"}
```

It contains no other metadata fields and saves neither media nor deduplication records. After authentication, return HTTP 200, Content-Type `application/json`, and:

```json
{"status":"ok"}
```

The client checks strict string status `ok`. This affects only the settings test result, not upload state. Network, timeout, DNS, or TLS errors show Unable to connect; 401/403 show Authentication failed; 404/405/422 or an invalid HTTP-200 response show that the URL is reachable but is not a Media Go receiver. Other responses show their HTTP status. HTTP and Content-Type are envelope checks; the JSON success condition is status alone.

## 9. Idempotency and conflicts

No extra Idempotency-Key or request UUID is used. Operation identities within each receiver are:

| Operation | Identity |
| --- | --- |
| `file` | media_id + file |
| `segment` | media_id + segment + segment.index |
| `finalize` | media_id + finalize |

Persist and freeze submitted metadata and media. Retries must not re-encode or rewrite files or reuse an identity for different content. JSON key order, whitespace, and multipart boundaries do not change request meaning. Filenames and file-part media types stay fixed after first submission.

The first successful save for an identity is authoritative. Persist the operation identity, metadata, and media association; memory-only deduplication is insufficient. Equivalent repeated metadata, filename, and media type return `{"status":"received"}` without duplicating or overwriting content, including concurrent retries. Retain acknowledged operation identities while retaining their media.

No SHA-256, other digest, or byte comparison is required. A receiver need not detect changed bytes under identical identity and metadata; it retains the first saved file. The client must keep bytes unchanged.

Changed annotations, time ranges, manifests, filenames, or media types under the same operation identity cause HTTP 409 / `idempotency_conflict`. Changed purpose, including empty versus nonempty, or changed delivery mode for one media ID is also a conflict.

After successful `finalize`, valid retries of saved requests still succeed. New segments outside the manifest cause 409 / `recording_finalized`. A failed finalization must not prematurely lock the recording. Empty annotations on old segment retries never overwrite final annotations.

## 10. Missing segments and failures

A finalization with missing files returns HTTP 409, for example:

```json
{"status":"error","error":{"code":"missing_segments","segment_indices":[1]}}
```

Indices must form a nonempty, unique array of nonnegative integers, excluding booleans. The client checks that each belongs to the fixed manifest and has an available local file, uploads only those segments, and retries the original `finalize`. Invalid or out-of-manifest indices are protocol errors and do not alter confirmations. Unrecoverable local files leave the item needing attention; the manifest is not reduced and all acknowledged segments are not resent.

Extra received segments, or indices or ranges that disagree with the manifest, cause HTTP 409 / `manifest_conflict` without requiring digest comparisons.

| Condition | Client behavior |
| --- | --- |
| Offline, timeout, 408/429, temporary 5xx | Retain data; retry with backoff and valid Retry-After |
| 409 / missing_segments | Upload declared missing segments, then retry finalization |
| 409 / idempotency_conflict, manifest_conflict, recording_finalized | Retain data, pause, show the conflict |
| 401/403, invalid URL/404/405, 413, 415, invalid metadata | Pause for correction; preserve request identity and content |
| HTTP 200 with invalid receipt | Retain data; report receipt error rather than success |

Parseable application errors use `status: error` and `error.code`. Invalid metadata or operations use HTTP 422. Proxies and transport layers may not return JSON; the client still retains data according to the transport result. See [HTTP conflict](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.10) and [Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html#section-10.2.3).

## 11. Queue, receivers, and background operation

Pending files, media IDs, button snapshots, annotation drafts, submitted metadata, and receipts survive reopening. Audio potentially needed for retries is not automatically cleaned up before the final save receipt.

Without a receiver, content stays local pending configuration. Receipts from different receivers cannot be combined into successful finalization. The first save receipt binds an item to the URL used for that request; subsequent requests use that receiver. Unacknowledged items follow the current configured URL. There is no redirect-to-another-receiver action. A permanently unavailable bound receiver leaves the item needing attention. Background responses retain the original request identity and URL regardless of the current screen or settings.

Credentials can be updated per receiver without changing content identity. HTTP redirects do not move uploads to a different receiver. Active recordings close segments according to their snapshots and send them promptly, but background scheduling after stopping may be delayed. Completion within seconds and continued capture after force quit are not promised.

Local retention, space checks, and item deletion follow the app's storage rules. Unuploaded content is not automatically deleted to free space. Deleting a local item does not recall receiver-side content.
