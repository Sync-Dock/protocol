# Sync Dock Protocol

Shared HTTP/JSON contracts for Sync Dock server and clients.

## Versioning

The current API namespace is `/v1`. Contract changes should be additive unless a coordinated server and client migration is planned.

## Authentication

Most endpoints use device-token authentication:

```http
Authorization: Bearer <device-token>
```

Device tokens are issued only when registering a user and first device. Clients must store tokens securely. The server stores only token hashes.

## Core Flows

### Register User And First Device

`POST /v1/auth/register`

Creates a user, a first device, and a sync cursor. The response returns the only plaintext copy of the device token.

### Register Folder

`POST /v1/folders`

Creates a user-owned sync folder and emits a durable `folder_created` event.

### Upload File Version

`POST /v1/uploads/prepare`

Creates or reuses an idempotent upload session and returns an S3-compatible presigned PUT URL.

Clients should include `base_version_id` when updating a known file version. If omitted or stale, the server may preserve the new upload as a conflict.

`POST /v1/uploads/{uploadID}/commit`

Verifies uploaded bytes by size and SHA-256 hash before creating blob metadata, file entry, file version, and a durable `file_version_created` event.

### Delete File

`DELETE /v1/folders/{folderID}/files?path=<relative-path>`

Creates a tombstone version and emits `file_deleted`. Clients must sync deletes through tombstones instead of inferring them from missing files.

### Poll Sync Events

`GET /v1/sync/events?limit=<n>`

Returns events after the authenticated device cursor and advances the cursor to the last returned event.

### Download File Version

`GET /v1/file-versions/{versionID}/download-url`

Returns an S3-compatible presigned GET URL for a file version owned by the authenticated user.

## Sync Invariants

- Paths are relative to a registered folder root.
- Paths must not be absolute, contain null bytes, or escape the folder with `..`.
- File contents are addressed by SHA-256 hashes.
- Deletions are explicit tombstone versions.
- Conflicting versions are preserved with `is_conflict=true`.
- Event ids are monotonically increasing per server database.
- Device cursors are advanced only after events are returned to that device.
