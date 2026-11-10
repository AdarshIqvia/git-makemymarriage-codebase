# Make My Marriage — API Documentation

## 1. Document status

- **Status:** Draft for review
- **Style:** JSON over HTTPS, Next.js route handlers under `/api`
- **Related documents:** `PRD.md`, `SYSTEM_DESIGN.md`, `DATABASE_DESIGN.md`

Every planner screen and guest screen uses these endpoints. There are no undocumented server actions, so each operation has one contract and one authorization check.

## 2. Conventions

### 2.1 Access levels

Each endpoint lists who may call it:

- **Public** — anyone, no session.
- **Household** — a guest with a valid household session cookie.
- **Manager** — an active Manager or Admin.
- **Admin** — an active Admin only.

Access is enforced on the server. Hiding a button in the interface is never the control.

### 2.2 Sessions

- **Planner session:** `mmm_session` cookie, set by `POST /api/auth/login`. HTTP-only, `Secure`, `SameSite=Lax`, 7-day lifetime, refreshed on use. Invalid once the user's `sessionVersion` changes or the account or membership is disabled.
- **Household session:** `mmm_guest` cookie, set by `POST /api/guest/session`. HTTP-only, `Secure`, `SameSite=Lax`, 30-day lifetime. Holds the household ID and a fingerprint of its `tokenHash`, so regenerating the link ends the session.
- A browser may hold both cookies. Photo endpoints act as the planner when a valid planner session is present.
- New Managers with `mustChangePassword` may call only `GET /api/auth/me`, `POST /api/auth/change-password`, and `POST /api/auth/logout`. Other planner endpoints return `403 PASSWORD_CHANGE_REQUIRED`.

### 2.3 Cross-site request protection

All `POST`, `PUT`, `PATCH`, and `DELETE` requests must send an `Origin` header matching the application URL. Requests from any other origin are rejected with `403 FORBIDDEN`. Together with `SameSite=Lax` cookies, this prevents other sites from submitting forms on a user's behalf.

### 2.4 Formats

- Request and response bodies are JSON (`Content-Type: application/json`), except file imports, which use `multipart/form-data`.
- IDs are strings.
- Dates are ISO 8601 in UTC, for example `2027-02-14T13:30:00.000Z`.
- Single resources are returned as an object. Lists are returned as:

```json
{
  "items": [],
  "nextCursor": "eyJwIjoiMjAyNy0wMi0xNFQxMzozMDowMFoiLCJpIjoiNjVm..."
}
```

### 2.5 Pagination

List endpoints accept `limit` (default 50, maximum 100; photo lists default 30) and `cursor`. Pass the previous response's `nextCursor` to fetch the next page. `nextCursor` is `null` on the last page. Cursors are opaque; clients must not build them.

### 2.6 Errors

All errors use one shape:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Attendee count must be at least 1 when the answer is Yes.",
    "details": [
      { "path": "attendeeCount", "message": "Must be at least 1" }
    ]
  }
}
```

Standard codes:

- `400 VALIDATION_FAILED` — the request failed Zod validation. `details` lists each field.
- `401 UNAUTHENTICATED` — no valid session.
- `401 INVALID_CREDENTIALS` — wrong email or password. Never says which.
- `401 INVALID_LINK` — the household link is unknown, regenerated, or disabled. Never says which.
- `403 FORBIDDEN` — signed in, but not allowed.
- `403 PASSWORD_CHANGE_REQUIRED` — a new Manager must change their password first.
- `404 NOT_FOUND` — the resource does not exist **or** the caller may not know it exists. A household asking about an event it is not invited to always gets `404`, never `403`.
- `409 CONFLICT` — the request conflicts with current state. A more specific code is used where listed.
- `413 PAYLOAD_TOO_LARGE` — file or body exceeds its limit.
- `415 UNSUPPORTED_MEDIA_TYPE` — file type not accepted.
- `422 IMAGE_REJECTED` — the uploaded file is not a valid image.
- `429 RATE_LIMITED` — too many requests. Includes a `Retry-After` header in seconds.
- `500 INTERNAL_ERROR` — unexpected failure. The message never includes internal details.

### 2.7 Caching

- Planner and household responses send `Cache-Control: private, no-store`.
- `GET /api/public/weddings/{slug}` and its cover image may be cached by the CDN.
- Signed photo URLs are valid until the end of the current one-hour window plus one hour, so the same URL is reused within a window and browsers can cache images.

### 2.8 Rate limits

Limits that use the database (sensitive actions):

- `POST /api/auth/login` — 5 per 15 minutes per email, 20 per 15 minutes per IP.
- `POST /api/auth/forgot-password` — 3 per hour per email, 10 per hour per IP.
- `POST /api/guest/session` — 20 per 15 minutes per IP.
- `POST /api/events/{eventId}/photos` — 60 per hour and 300 per day per household.
- `GET /api/admin/exports/sending-sheet` — 10 per hour per user.

All other endpoints rely on general Vercel firewall limits per IP.

## 3. Planner authentication

### `POST /api/auth/login`

Access: Public

Request:

```json
{ "email": "priya@example.com", "password": "correct horse battery staple" }
```

Response `200`, sets `mmm_session`:

```json
{
  "user": { "id": "665f...", "email": "priya@example.com", "displayName": "Priya" },
  "role": "ADMIN",
  "mustChangePassword": false
}
```

Errors: `400 VALIDATION_FAILED`, `401 INVALID_CREDENTIALS`, `429 RATE_LIMITED`.

Disabled accounts also receive `401 INVALID_CREDENTIALS`.

### `POST /api/auth/logout`

Access: any planner session. Response `204`, clears `mmm_session`.

### `GET /api/auth/me`

Access: any planner session.

Response `200`:

```json
{
  "user": { "id": "665f...", "email": "priya@example.com", "displayName": "Priya" },
  "role": "ADMIN",
  "mustChangePassword": false
}
```

Errors: `401 UNAUTHENTICATED`.

### `POST /api/auth/change-password`

Access: any planner session.

Request:

```json
{ "currentPassword": "old", "newPassword": "at-least-12-characters" }
```

Response `204`. Increments `sessionVersion`, signs out other sessions, keeps the current one, and clears `mustChangePassword`.

Errors: `400 VALIDATION_FAILED` (new password under 12 characters), `401 INVALID_CREDENTIALS` (current password wrong).

### `POST /api/auth/forgot-password`

Access: Public. Requires the email provider to be configured.

Request:

```json
{ "email": "priya@example.com" }
```

Response `202` whether or not the account exists. If it exists and is active, a reset link valid for 30 minutes is emailed.

Errors: `400 VALIDATION_FAILED`, `429 RATE_LIMITED`.

### `POST /api/auth/reset-password`

Access: Public.

Request:

```json
{ "token": "from-the-email-link", "newPassword": "at-least-12-characters" }
```

Response `204`. Marks the token used, increments `sessionVersion`, and clears `mustChangePassword`.

Errors: `400 VALIDATION_FAILED`, `400 RESET_LINK_INVALID` (unknown, expired, or already used).

## 4. Planner members

### `GET /api/admin/members`

Access: Admin.

Response `200`:

```json
{
  "items": [
    {
      "membershipId": "6660...",
      "userId": "665f...",
      "email": "rahul@example.com",
      "displayName": "Rahul",
      "role": "MANAGER",
      "status": "ACTIVE",
      "lastLoginAt": "2027-01-10T08:00:00.000Z"
    }
  ],
  "nextCursor": null
}
```

### `POST /api/admin/members`

Access: Admin. Creates a planner account and membership.

Request:

```json
{
  "email": "rahul@example.com",
  "displayName": "Rahul",
  "role": "MANAGER",
  "temporaryPassword": "share-this-privately"
}
```

Response `201`: the member object. The new user must change the password at first sign-in.

Errors: `400 VALIDATION_FAILED`, `409 EMAIL_ALREADY_EXISTS`.

Audited as `member.created`.

### `PATCH /api/admin/members/{membershipId}`

Access: Admin.

Request (any subset):

```json
{ "role": "ADMIN", "status": "DISABLED" }
```

Response `200`: the updated member. Disabling a member signs them out everywhere.

Errors: `404 NOT_FOUND`, `409 LAST_ADMIN` (would leave no active Admin), `409 CANNOT_DISABLE_SELF`.

Audited as `member.updated`.

### `POST /api/admin/members/{membershipId}/reset-password`

Access: Admin. Sets a new temporary password for a Manager, for use when email recovery is not configured.

Request:

```json
{ "temporaryPassword": "share-this-privately" }
```

Response `204`. Sets `mustChangePassword` and signs the user out everywhere.

Errors: `404 NOT_FOUND`, `403 FORBIDDEN` (target is an Admin other than the caller).

Audited as `member.password_reset`.

## 5. Wedding

### `GET /api/admin/wedding`

Access: Manager.

Response `200`:

```json
{
  "id": "6650...",
  "coupleNames": "Priya & Arjun",
  "slug": "priya-arjun",
  "story": "We met at college...",
  "welcomeMessage": "We would love to celebrate with you.",
  "city": "Jaipur",
  "startDate": "2027-02-12T00:00:00.000Z",
  "endDate": "2027-02-14T00:00:00.000Z",
  "timezone": "Asia/Kolkata",
  "contactInfo": { "name": "Rahul", "phone": "+919800000000", "email": null },
  "coverImageUrl": "/api/public/weddings/priya-arjun/cover",
  "inviteMessageTemplate": "Dear {contactName}, {coupleNames} invite you to their wedding. Your invitation: {link}",
  "highAttendanceThreshold": 15,
  "status": "ACTIVE"
}
```

### `PATCH /api/admin/wedding`

Access: Admin.

Request: any subset of the editable fields above except `id` and `coverImageUrl`.

Response `200`: the updated wedding. Refreshes the cached public page.

Errors: `400 VALIDATION_FAILED`, `409 SLUG_TAKEN`.

### `POST /api/admin/wedding/cover`

Access: Admin. Starts a cover-image upload.

Request:

```json
{ "contentType": "image/jpeg", "bytes": 2483021 }
```

Response `200`:

```json
{
  "uploadUrl": "https://<account>.r2.cloudflarestorage.com/...",
  "uploadHeaders": { "Content-Type": "image/jpeg" },
  "expiresAt": "2027-01-10T08:10:00.000Z"
}
```

The browser uploads the file with `PUT uploadUrl`, then calls the finalize endpoint.

Errors: `413 PAYLOAD_TOO_LARGE` (over 10 MB), `415 UNSUPPORTED_MEDIA_TYPE`.

### `POST /api/admin/wedding/cover/finalize`

Access: Admin. Validates and optimizes the uploaded cover image.

Response `200`: the updated wedding.

Errors: `404 NOT_FOUND` (nothing uploaded), `422 IMAGE_REJECTED`.

## 6. Events

Event object:

```json
{
  "id": "6651...",
  "name": "Sangeet",
  "description": "An evening of music and dance.",
  "startAt": "2027-02-12T13:30:00.000Z",
  "endAt": "2027-02-12T18:30:00.000Z",
  "venueName": "Rambagh Palace",
  "venueAddress": "Bhawani Singh Road, Jaipur",
  "mapUrl": "https://maps.google.com/?q=...",
  "livestreamUrl": "https://youtube.com/live/...",
  "sortOrder": 2,
  "status": "ACTIVE"
}
```

### `GET /api/admin/events`

Access: Manager. Query: `status` (`ACTIVE` default, `ARCHIVED`, or `ALL`).

Response `200`: list of events, each with summary counts:

```json
{
  "items": [
    {
      "id": "6651...",
      "name": "Sangeet",
      "startAt": "2027-02-12T13:30:00.000Z",
      "status": "ACTIVE",
      "sortOrder": 2,
      "counts": {
        "invitedHouseholds": 640,
        "yes": { "households": 410, "attendees": 1380 },
        "maybe": { "households": 35, "attendees": 90 },
        "no": { "households": 60 },
        "pending": { "households": 135 },
        "photos": 212
      }
    }
  ],
  "nextCursor": null
}
```

### `POST /api/admin/events`

Access: Manager.

Request:

```json
{
  "name": "Sangeet",
  "description": "An evening of music and dance.",
  "startAt": "2027-02-12T13:30:00.000Z",
  "endAt": "2027-02-12T18:30:00.000Z",
  "venueName": "Rambagh Palace",
  "venueAddress": "Bhawani Singh Road, Jaipur",
  "mapUrl": "https://maps.google.com/?q=...",
  "livestreamUrl": null
}
```

Only `name` and `startAt` are required. The event is added at the end of the order.

Response `201`: the event.

Errors: `400 VALIDATION_FAILED` (for example, `endAt` before `startAt`, or a non-https URL).

### `GET /api/admin/events/{eventId}`

Access: Manager. Response `200`: the event with `counts`. Errors: `404 NOT_FOUND`.

### `PATCH /api/admin/events/{eventId}`

Access: Manager. Request: any subset of event fields, including `status` to archive or restore. Set `livestreamUrl` to `null` to remove it.

Response `200`: the updated event. Errors: `400 VALIDATION_FAILED`, `404 NOT_FOUND`.

### `PUT /api/admin/events/order`

Access: Manager. Sets the display order.

Request:

```json
{ "eventIds": ["6651...", "6652...", "6653..."] }
```

Must list every active event exactly once. Response `204`.

Errors: `400 VALIDATION_FAILED` (missing, duplicate, or unknown IDs).

## 7. Households

Household object (planner view):

```json
{
  "id": "6654...",
  "label": "Sharma family",
  "primaryContactName": "Anil Sharma",
  "phone": "+919811111111",
  "email": "anil@example.com",
  "memberNames": ["Anil", "Sunita", "Rohan"],
  "internalNotes": "Arriving from Delhi",
  "status": "ACTIVE",
  "delivery": { "status": "SENT", "markedAt": "2027-01-05T10:00:00.000Z", "markedBy": "Rahul" },
  "linkRegeneratedAt": "2026-12-20T09:00:00.000Z",
  "invitations": [
    {
      "invitationId": "6655...",
      "eventId": "6651...",
      "eventName": "Sangeet",
      "response": "YES",
      "attendeeCount": 4,
      "attendeeNames": [],
      "respondedAt": "2027-01-06T12:00:00.000Z",
      "flagged": false
    }
  ]
}
```

`flagged` is `true` when `attendeeCount` exceeds the wedding's `highAttendanceThreshold`.

### `GET /api/admin/households`

Access: Manager.

Query:

- `q` — prefix search on label and contact name, or exact match on phone or email.
- `status` — `ACTIVE` (default), `DISABLED`, or `ALL`.
- `eventId` — only households invited to this event.
- `response` — with `eventId`: only households with this answer for that event.
- `delivery` — `SENT` or `NOT_SENT`.
- `flagged` — `true` for households with any flagged count.
- `limit`, `cursor`.

Response `200`: list of households. Each list item includes `invitations` without `attendeeNames`.

### `POST /api/admin/households`

Access: Manager. Creates a household and, optionally, its invitations in one transaction.

Request:

```json
{
  "label": "Sharma family",
  "primaryContactName": "Anil Sharma",
  "phone": "9811111111",
  "email": "anil@example.com",
  "memberNames": ["Anil", "Sunita", "Rohan"],
  "internalNotes": "Arriving from Delhi",
  "eventIds": ["6651...", "6652..."]
}
```

Only `label` and `primaryContactName` are required.

Response `201`: the household, plus `possibleDuplicates` listing existing households with the same phone or email. Duplicates do not block creation.

Errors: `400 VALIDATION_FAILED`, `404 NOT_FOUND` (unknown event).

### `GET /api/admin/households/{householdId}`

Access: Manager. Response `200`: the full household object. Errors: `404 NOT_FOUND`.

### `PATCH /api/admin/households/{householdId}`

Access: Manager. Request: any subset of `label`, `primaryContactName`, `phone`, `email`, `memberNames`, `internalNotes`, `status`.

Disabling a household ends its guest sessions and makes its link return `401 INVALID_LINK`. Its RSVPs remain in reports, marked as disabled.

Response `200`: the updated household. Errors: `400 VALIDATION_FAILED`, `404 NOT_FOUND`.

Status changes are audited.

### `PATCH /api/admin/households/delivery`

Access: Manager. Bulk-marks invitations as sent or not sent.

Request:

```json
{ "householdIds": ["6654...", "6656..."], "status": "SENT" }
```

Maximum 500 IDs. Response `200`:

```json
{ "updated": 2 }
```

## 8. Private invitation links

### `GET /api/admin/households/{householdId}/invite-link`

Access: Manager. Returns the household's current link. The same link is returned every time until it is regenerated.

Response `200`:

```json
{
  "url": "https://makemymarriage.example/i/4hX2...",
  "message": "Dear Anil Sharma, Priya & Arjun invite you to their wedding. Your invitation: https://makemymarriage.example/i/4hX2...",
  "whatsappUrl": "https://wa.me/919811111111?text=Dear%20Anil%20Sharma...",
  "emailSubject": "Invitation from Priya & Arjun",
  "emailBody": "Dear Anil Sharma, ..."
}
```

`whatsappUrl` is `null` when the household has no phone number. The planner's browser draws the QR code and invitation card from `url`.

Errors: `404 NOT_FOUND`.

### `POST /api/admin/households/{householdId}/invite-link/regenerate`

Access: Manager. Replaces the link. The old link and any guest sessions using it stop working immediately.

Response `200`: the same shape as the `GET` above, with the new link.

Errors: `404 NOT_FOUND`. Audited as `household.link.regenerated`.

## 9. Imports and exports

### `POST /api/admin/households/import/preview`

Access: Manager. Validates a CSV or Excel file without saving anything.

Request: `multipart/form-data` with field `file`. Maximum 2 MB and 2,000 rows.

Expected columns (header names are case-insensitive): `label`, `contact_name`, `phone`, `email`, `member_names` (separated by `;`), `notes`.

Response `200`:

```json
{
  "batchId": "b7c1e2d4-5f60-4a1b-9c3d-2e8f7a6b5c4d",
  "summary": { "rows": 1250, "valid": 1238, "invalid": 12, "possibleDuplicates": 40 },
  "rows": [
    {
      "rowNumber": 2,
      "data": {
        "label": "Sharma family",
        "primaryContactName": "Anil Sharma",
        "phone": "+919811111111",
        "email": "anil@example.com",
        "memberNames": ["Anil", "Sunita"],
        "internalNotes": null
      },
      "errors": [],
      "duplicates": [
        { "source": "EXISTING", "householdId": "6654...", "label": "Sharma family", "matchedOn": "phone" },
        { "source": "FILE", "rowNumber": 87, "matchedOn": "phone" }
      ]
    }
  ]
}
```

Spreadsheet formulas are read as plain text and never evaluated.

Errors: `413 PAYLOAD_TOO_LARGE`, `415 UNSUPPORTED_MEDIA_TYPE`, `400 IMPORT_TOO_MANY_ROWS`, `400 IMPORT_MISSING_COLUMNS`.

### `POST /api/admin/households/import/commit`

Access: Manager. Saves the rows the planner confirmed from the preview.

Request:

```json
{
  "batchId": "b7c1e2d4-5f60-4a1b-9c3d-2e8f7a6b5c4d",
  "rows": [
    {
      "rowNumber": 2,
      "label": "Sharma family",
      "primaryContactName": "Anil Sharma",
      "phone": "+919811111111",
      "email": "anil@example.com",
      "memberNames": ["Anil", "Sunita"],
      "internalNotes": null
    }
  ],
  "eventIds": ["6651...", "6652..."]
}
```

Rows are validated again on the server. `eventIds` optionally invites every imported household to those events. Maximum 2,000 rows.

If the request fails partway, resend it unchanged. Rows already created under the same `batchId` and `rowNumber` are skipped, so nobody is imported twice.

Response `200`:

```json
{ "created": 1238, "alreadyCreated": 0, "invitationsCreated": 2476 }
```

Errors: `400 VALIDATION_FAILED` (lists every invalid row; nothing is saved until all rows pass validation). Audited as `households.imported`.

### `GET /api/admin/exports/households`

Access: Manager. Returns `text/csv`: label, contact name, phone, email, member names, status, delivery status, and one column per event showing the household's answer and count. No links.

### `GET /api/admin/exports/rsvps`

Access: Manager. Query: optional `eventId`. Returns `text/csv`: one row per invitation with household, event, answer, count, attendee names, and last update.

### `GET /api/admin/exports/sending-sheet`

Access: Admin. Query: same filters as `GET /api/admin/households`.

Returns `text/csv` with: label, contact name, phone, private link, message, and WhatsApp link. Family members can split the rows and send them by hand.

Errors: `429 RATE_LIMITED`. Audited as `export.sending_sheet` with the row count.

### `GET /api/admin/exports/tasks`

Access: Manager. Returns `text/csv` of all tasks.

## 10. Event invitations and RSVPs (planner)

### `PUT /api/admin/events/{eventId}/invitations`

Access: Manager. Adds or removes households from an event.

Request:

```json
{
  "add": ["6654...", "6656..."],
  "remove": ["6657..."],
  "force": false
}
```

Maximum 1,000 IDs in total. Adding an already-invited household is ignored. Removing a household that has already replied requires `"force": true`; its RSVP history is kept, but it loses access to the event, its livestream link, and its album.

Response `200`:

```json
{ "added": 2, "removed": 1, "alreadyInvited": 0 }
```

Errors: `404 NOT_FOUND` (event), `409 HAS_RESPONSES` (lists households with replies when `force` is false):

```json
{
  "error": {
    "code": "HAS_RESPONSES",
    "message": "1 household has already replied to this event.",
    "details": [{ "householdId": "6657...", "label": "Verma family", "response": "YES" }]
  }
}
```

### `GET /api/admin/events/{eventId}/rsvps`

Access: Manager. Query: `response`, `q`, `flagged`, `limit`, `cursor`.

Response `200`:

```json
{
  "totals": {
    "invitedHouseholds": 640,
    "yes": { "households": 410, "attendees": 1380 },
    "maybe": { "households": 35, "attendees": 90 },
    "no": { "households": 60 },
    "pending": { "households": 135 }
  },
  "items": [
    {
      "invitationId": "6655...",
      "householdId": "6654...",
      "label": "Sharma family",
      "primaryContactName": "Anil Sharma",
      "response": "YES",
      "attendeeCount": 4,
      "attendeeNames": ["Anil", "Sunita"],
      "respondedAt": "2027-01-06T12:00:00.000Z",
      "lastChangedBy": { "type": "HOUSEHOLD" },
      "flagged": false
    }
  ],
  "nextCursor": null
}
```

### `PATCH /api/admin/invitations/{invitationId}`

Access: Manager. Records an answer on the household's behalf, for example after a phone call.

Request: the same body and rules as the guest RSVP endpoint (section 13).

Response `200`: the updated invitation. A history entry is written with `changedBy.type = USER`.

Errors: `400 VALIDATION_FAILED`, `404 NOT_FOUND`. Audited as `rsvp.updated_by_planner`.

### `GET /api/admin/invitations/{invitationId}/history`

Access: Manager.

Response `200`:

```json
{
  "items": [
    {
      "from": { "response": "YES", "attendeeCount": 4 },
      "to": { "response": "NO", "attendeeCount": 0 },
      "changedBy": { "type": "HOUSEHOLD" },
      "changedAt": "2027-01-20T16:00:00.000Z"
    },
    {
      "from": { "response": "PENDING", "attendeeCount": 0 },
      "to": { "response": "YES", "attendeeCount": 4 },
      "changedBy": { "type": "USER", "displayName": "Rahul" },
      "changedAt": "2027-01-06T12:00:00.000Z"
    }
  ],
  "nextCursor": null
}
```

### `GET /api/admin/rsvp-changes`

Access: Manager. Recent RSVP changes across the wedding.

Query:

- `type` — `ALL` (default) or `CANCELLATIONS` (Yes changed to No).
- `eventId`, `since` (ISO date), `limit`, `cursor`.

Response `200`: list of changes, each with `householdId`, `label`, `eventId`, `eventName`, `from`, `to`, `changedBy`, and `changedAt`.

## 11. Dashboard

### `GET /api/admin/dashboard`

Access: Manager.

Response `200`:

```json
{
  "upcomingEvents": [
    { "id": "6651...", "name": "Sangeet", "startAt": "2027-02-12T13:30:00.000Z", "yesAttendees": 1380 }
  ],
  "households": { "active": 1240, "disabled": 8 },
  "delivery": { "sent": 1100, "notSent": 140 },
  "rsvp": {
    "byEvent": [
      {
        "eventId": "6651...",
        "eventName": "Sangeet",
        "yes": { "households": 410, "attendees": 1380 },
        "maybe": { "households": 35, "attendees": 90 },
        "no": { "households": 60 },
        "pending": { "households": 135 }
      }
    ],
    "householdsAwaitingAnyReply": 180,
    "flaggedCounts": 6,
    "recentCancellations": 4
  },
  "tasks": { "todo": 22, "inProgress": 9, "done": 61, "overdue": 3 },
  "photos": { "published": 1840, "hidden": 3, "uploadedLast24h": 120 }
}
```

Each section is computed independently. If one fails, it is returned as `null` and the others still load.

## 12. Tasks

Task object:

```json
{
  "id": "6658...",
  "title": "Confirm caterer headcount",
  "description": "Send Sangeet numbers by 5 Feb",
  "eventId": "6651...",
  "assignee": { "userId": "665f...", "displayName": "Rahul" },
  "priority": "HIGH",
  "status": "TODO",
  "dueAt": "2027-02-05T06:30:00.000Z",
  "completedAt": null,
  "overdue": false,
  "createdBy": { "userId": "665e...", "displayName": "Priya" },
  "createdAt": "2027-01-02T10:00:00.000Z"
}
```

### `GET /api/admin/tasks`

Access: Manager. Query: `status`, `assigneeUserId`, `eventId`, `priority`, `overdue=true`, `q`, `limit`, `cursor`. Sorted by due date, with tasks without a due date last.

### `POST /api/admin/tasks`

Access: Manager. Request: `title` (required), `description`, `eventId`, `assigneeUserId`, `priority`, `status`, `dueAt`. Response `201`: the task.

Errors: `400 VALIDATION_FAILED`, `404 NOT_FOUND` (unknown event or assignee).

### `PATCH /api/admin/tasks/{taskId}`

Access: Manager. Request: any subset of task fields. Moving to `DONE` sets `completedAt`; leaving `DONE` clears it. Response `200`: the task.

### `DELETE /api/admin/tasks/{taskId}`

Access: Manager. Response `204`. Audited as `task.deleted`.

## 13. Guest invitation and RSVP

### `POST /api/guest/session`

Access: Public. Called by the `/i/{token}` page when a guest opens their link or scans their QR code.

Request:

```json
{ "token": "4hX2..." }
```

Response `200`, sets `mmm_guest`:

```json
{ "household": { "label": "Sharma family", "primaryContactName": "Anil Sharma" } }
```

Errors: `401 INVALID_LINK`, `429 RATE_LIMITED`.

### `DELETE /api/guest/session`

Access: Household. Response `204`, clears `mmm_guest`.

### `GET /api/guest/invitation`

Access: Household. Everything the guest page needs in one request.

Response `200`:

```json
{
  "household": {
    "label": "Sharma family",
    "primaryContactName": "Anil Sharma",
    "memberNames": ["Anil", "Sunita", "Rohan"]
  },
  "wedding": {
    "coupleNames": "Priya & Arjun",
    "welcomeMessage": "We would love to celebrate with you.",
    "story": "We met at college...",
    "city": "Jaipur",
    "timezone": "Asia/Kolkata",
    "contactInfo": { "name": "Rahul", "phone": "+919800000000" },
    "coverImageUrl": "/api/public/weddings/priya-arjun/cover"
  },
  "events": [
    {
      "id": "6651...",
      "name": "Sangeet",
      "description": "An evening of music and dance.",
      "startAt": "2027-02-12T13:30:00.000Z",
      "endAt": "2027-02-12T18:30:00.000Z",
      "venueName": "Rambagh Palace",
      "venueAddress": "Bhawani Singh Road, Jaipur",
      "mapUrl": "https://maps.google.com/?q=...",
      "livestreamUrl": "https://youtube.com/live/...",
      "rsvp": {
        "response": "YES",
        "attendeeCount": 4,
        "attendeeNames": ["Anil", "Sunita"],
        "respondedAt": "2027-01-06T12:00:00.000Z"
      }
    }
  ]
}
```

Only active events the household is invited to are included, in display order. `livestreamUrl` is `null` when none is set. Internal notes, delivery status, and other households are never included.

Errors: `401 UNAUTHENTICATED` (no session), `401 INVALID_LINK` (link regenerated or household disabled since the session started).

### `PUT /api/guest/events/{eventId}/rsvp`

Access: Household. Saves the household's answer for one event. Repeating the same request is harmless.

Request:

```json
{ "response": "YES", "attendeeCount": 4, "attendeeNames": ["Anil", "Sunita"] }
```

Rules:

- `response` is `YES`, `NO`, or `MAYBE`.
- `YES` requires `attendeeCount` of at least 1.
- `NO` ignores `attendeeCount` and `attendeeNames`, storing 0 and an empty list.
- `MAYBE` accepts `attendeeCount` of 0 or more.
- `attendeeCount` is a whole number up to 9,999.
- `attendeeNames` is optional, each name up to 100 characters, and the list may not be longer than `attendeeCount`.
- The latest save wins.

Response `200`:

```json
{
  "eventId": "6651...",
  "response": "YES",
  "attendeeCount": 4,
  "attendeeNames": ["Anil", "Sunita"],
  "respondedAt": "2027-01-06T12:00:00.000Z"
}
```

Errors: `400 VALIDATION_FAILED`, `401 INVALID_LINK`, `404 NOT_FOUND` (not invited, or event archived).

## 14. Photos

Planners and households use the same photo endpoints. The server decides what the caller may do from their session.

Photo object:

```json
{
  "id": "6659...",
  "eventId": "6651...",
  "status": "PUBLISHED",
  "width": 2560,
  "height": 1707,
  "thumbUrl": "https://<account>.r2.cloudflarestorage.com/...signed...",
  "displayUrl": "https://<account>.r2.cloudflarestorage.com/...signed...",
  "downloadUrl": "https://<account>.r2.cloudflarestorage.com/...signed...&response-content-disposition=attachment",
  "urlsExpireAt": "2027-02-12T15:00:00.000Z",
  "publishedAt": "2027-02-12T14:05:00.000Z",
  "uploadedByMe": true,
  "canDelete": true
}
```

Planners additionally receive `uploader` (household label or planner name) and see `HIDDEN` photos. Households do not see who uploaded other photos.

### `GET /api/events/{eventId}/photos`

Access: Household invited to the event, or Manager.

Query: `limit` (default 30, maximum 60), `cursor`, `mine=true` (only the caller's uploads). Planners may also pass `status=HIDDEN`.

Response `200`: list of photo objects, newest first.

Errors: `401 UNAUTHENTICATED`, `404 NOT_FOUND` (event unknown, archived for guests, or household not invited).

### `POST /api/events/{eventId}/photos`

Access: Household invited to the event, or Manager. Starts one photo upload.

The browser resizes the photo to about 2,560 pixels on the long edge and converts it to JPEG before calling this endpoint.

Request:

```json
{ "contentType": "image/jpeg", "bytes": 1843200 }
```

Accepted types: `image/jpeg`, `image/png`, `image/webp`. Maximum 10 MB.

Response `201`:

```json
{
  "photoId": "6659...",
  "uploadUrl": "https://<account>.r2.cloudflarestorage.com/...signed-put...",
  "uploadHeaders": { "Content-Type": "image/jpeg", "Content-Length": "1843200" },
  "expiresAt": "2027-02-12T14:15:00.000Z"
}
```

The browser sends the file with `PUT uploadUrl` and exactly those headers, then calls finalize. Upload URLs are valid for 10 minutes.

Errors: `404 NOT_FOUND`, `413 PAYLOAD_TOO_LARGE`, `415 UNSUPPORTED_MEDIA_TYPE`, `429 RATE_LIMITED`.

### `POST /api/photos/{photoId}/finalize`

Access: the household or planner that started the upload.

Validates the uploaded file, removes location data and other metadata, creates the thumbnail and display images, deletes the temporary upload, and publishes the photo.

Response `200`: the photo object with `status: "PUBLISHED"`.

Safe to repeat:

- If the photo is already published, returns it unchanged.
- If another finalize call is still processing it, returns `202` with `{ "status": "PROCESSING" }`. The client retries after two seconds.

Errors: `404 NOT_FOUND` (unknown photo or not the uploader), `409 UPLOAD_MISSING` (no file was uploaded), `413 PAYLOAD_TOO_LARGE` (uploaded file larger than declared), `422 IMAGE_REJECTED` (not a valid image, or larger than 50 megapixels).

### `DELETE /api/photos/{photoId}`

Access: the household that uploaded the photo, or Manager.

Response `204`. The photo disappears immediately; its files are removed by the daily job.

Errors: `404 NOT_FOUND` (unknown photo, or a household that is not the uploader — other households cannot learn the photo exists). Planner deletions are audited as `photo.deleted`.

### `PATCH /api/admin/photos/{photoId}`

Access: Manager. Hides or restores a photo.

Request:

```json
{ "status": "HIDDEN" }
```

`status` is `HIDDEN` or `PUBLISHED`. Response `200`: the photo object. Errors: `404 NOT_FOUND`, `409 CONFLICT` (photo not in a published or hidden state). Audited as `photo.hidden` or `photo.restored`.

### `GET /api/admin/photos`

Access: Manager. Moderation feed across all events.

Query: `eventId`, `status` (`PUBLISHED`, `HIDDEN`, or `FAILED`), `householdId`, `limit`, `cursor`.

Response `200`: list of photo objects with `uploader`, newest first.

## 15. Public

### `GET /api/public/weddings/{slug}`

Access: Public. Cacheable.

Response `200`:

```json
{
  "coupleNames": "Priya & Arjun",
  "story": "We met at college...",
  "welcomeMessage": "We would love to celebrate with you.",
  "city": "Jaipur",
  "startDate": "2027-02-12T00:00:00.000Z",
  "endDate": "2027-02-14T00:00:00.000Z",
  "contactInfo": { "name": "Rahul", "phone": "+919800000000" },
  "coverImageUrl": "/api/public/weddings/priya-arjun/cover"
}
```

No events, guests, livestream links, or photos are included.

Errors: `404 NOT_FOUND` (unknown slug or wedding still in `DRAFT`).

### `GET /api/public/weddings/{slug}/cover`

Access: Public. Streams the optimized cover image with a long CDN cache lifetime. Errors: `404 NOT_FOUND`.

## 16. Internal

### `GET /api/internal/cron/maintenance`

Access: Vercel Cron only. Requires `Authorization: Bearer {CRON_SECRET}`; any other caller gets `401`.

Runs once a day and:

- Marks photos stuck in `UPLOADING` for 24 hours as `FAILED` and removes their temporary files.
- Removes R2 files for photos marked `DELETED` and sets `purgedAt`.
- Exports core collections to the private R2 backup prefix and keeps the last seven exports.

Each step processes a bounded batch and continues on the next run.

Response `200`:

```json
{ "failedUploads": 3, "purgedPhotos": 12, "backupKey": "backups/2027-02-13.json.gz", "durationMs": 41250 }
```

### `GET /api/health`

Access: Public. Returns `200 { "status": "ok" }` without touching the database. Used by uptime checks.

## 17. Endpoint index

```text
Auth
  POST   /api/auth/login                                   Public
  POST   /api/auth/logout                                  Planner
  GET    /api/auth/me                                      Planner
  POST   /api/auth/change-password                         Planner
  POST   /api/auth/forgot-password                         Public
  POST   /api/auth/reset-password                          Public

Members
  GET    /api/admin/members                                Admin
  POST   /api/admin/members                                Admin
  PATCH  /api/admin/members/{membershipId}                 Admin
  POST   /api/admin/members/{membershipId}/reset-password  Admin

Wedding
  GET    /api/admin/wedding                                Manager
  PATCH  /api/admin/wedding                                Admin
  POST   /api/admin/wedding/cover                          Admin
  POST   /api/admin/wedding/cover/finalize                 Admin

Events
  GET    /api/admin/events                                 Manager
  POST   /api/admin/events                                 Manager
  GET    /api/admin/events/{eventId}                       Manager
  PATCH  /api/admin/events/{eventId}                       Manager
  PUT    /api/admin/events/order                           Manager

Households and links
  GET    /api/admin/households                             Manager
  POST   /api/admin/households                             Manager
  GET    /api/admin/households/{householdId}               Manager
  PATCH  /api/admin/households/{householdId}               Manager
  PATCH  /api/admin/households/delivery                    Manager
  GET    /api/admin/households/{householdId}/invite-link   Manager
  POST   /api/admin/households/{householdId}/invite-link/regenerate  Manager

Imports and exports
  POST   /api/admin/households/import/preview              Manager
  POST   /api/admin/households/import/commit               Manager
  GET    /api/admin/exports/households                     Manager
  GET    /api/admin/exports/rsvps                          Manager
  GET    /api/admin/exports/sending-sheet                  Admin
  GET    /api/admin/exports/tasks                          Manager

Invitations and RSVPs (planner)
  PUT    /api/admin/events/{eventId}/invitations           Manager
  GET    /api/admin/events/{eventId}/rsvps                 Manager
  PATCH  /api/admin/invitations/{invitationId}             Manager
  GET    /api/admin/invitations/{invitationId}/history     Manager
  GET    /api/admin/rsvp-changes                           Manager

Dashboard
  GET    /api/admin/dashboard                              Manager

Tasks
  GET    /api/admin/tasks                                  Manager
  POST   /api/admin/tasks                                  Manager
  PATCH  /api/admin/tasks/{taskId}                         Manager
  DELETE /api/admin/tasks/{taskId}                         Manager

Guest
  POST   /api/guest/session                                Public
  DELETE /api/guest/session                                Household
  GET    /api/guest/invitation                             Household
  PUT    /api/guest/events/{eventId}/rsvp                  Household

Photos
  GET    /api/events/{eventId}/photos                      Household (invited) or Manager
  POST   /api/events/{eventId}/photos                      Household (invited) or Manager
  POST   /api/photos/{photoId}/finalize                    Uploader
  DELETE /api/photos/{photoId}                             Uploader household or Manager
  PATCH  /api/admin/photos/{photoId}                       Manager
  GET    /api/admin/photos                                 Manager

Public and internal
  GET    /api/public/weddings/{slug}                       Public
  GET    /api/public/weddings/{slug}/cover                 Public
  GET    /api/internal/cron/maintenance                    Vercel Cron
  GET    /api/health                                       Public
```

## 18. Open decisions

- **Email provider for password reset.** Until one is chosen, `POST /api/auth/forgot-password` returns `503 EMAIL_NOT_CONFIGURED`, and Admins reset Manager passwords with the member reset endpoint.
- **Attendance flag threshold.** Defaults to 15 and can be changed in the wedding settings.
- **Upload limits per household.** Proposed at 60 an hour and 300 a day; adjust after the pilot.
