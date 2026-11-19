# DrugTime API Contract

API specifications and contracts for DrugTime.

## Source of truth

`openapi.json` is the hand-written OpenAPI 3.1 contract shared by the backend and
clients. `openapi.generated.json` is an ignored comparison artifact, not a
replacement for this contract. Authentication, medication matching, and all other
existing operations retain their current definitions.

## Linked patients for the family patient switcher

`GET /family/patients` supplies PB22's patient switcher. It requires an app session
bearer token and returns `items` containing only `patient_id` and nullable
`patient_full_name`. Each patient must have an active link to the caller, sharing
enabled, and current `family_sharing` and `health_data` consent. The caller's own
consents do not grant or remove access to another patient's data.

No query parameters or body are accepted (422); there is no pagination or silent
truncation. Names sort by Unicode code point ascending, null last, with patient ID
as the tie-breaker. No eligible patients returns 200 with an empty list. Admins
receive 403 `khong_co_quyen`; missing or invalid tokens receive 401.

Permissions are checked from the database on every call. Successful patient reads
are audited per returned patient under `danh_sach_benh_nhan_lien_ket`; empty lists
and admin denials use request-level metadata audit without fabricated patient IDs.
Audit or database failure returns 503 `service_unavailable`, never a partial list.
Allowed access logs for one request commit together; an audit failure rolls back
all of that request's allowed entries. Denied audit uses its separate transaction.
All responses carry `Cache-Control: no-store`. This list does not grant permission
to subsequent health requests: those endpoints recheck their own authorization.

## Health profile and conditions

| Method and path | Purpose | Success |
|---|---|---|
| `GET /profile` | Read the caller's profile and condition links | `200` |
| `PUT /profile` | Update personal fields and optionally the login email | `204`, no body |
| `GET /conditions` | Read the shared condition catalog | `200` |
| `POST /profile/conditions` | Add a condition link for the caller | `201` |
| `PATCH /profile/conditions/{id}` | Update a link's `subtype` and/or `stage` | `200` |
| `DELETE /profile/conditions/{id}` | Hard delete the caller's link | `204`, no body |

These operations require bearer authentication. Profile operations are self-only:
`patient_id` and `user_id` query selectors return `422 profile_is_self_only`.
Current `health_data` consent, PEP authorization and durable access audit are
required for profile reads/writes; authorization or audit failure cannot return a
successful health response. Admin status does not grant access to health data.
`GET /conditions` returns a shared catalog, not patient health data, and does not
require health-data consent. Its `limit` defaults to 20 (1–100), and `offset` to 0.

`PUT /profile` requires `full_name`, `date_of_birth` and `gender`; it leaves condition
links unchanged. Omit `email` to preserve it; `null` is invalid. A supplied valid
email updates the login address without sending OTP. This operation is separate
from onboarding's existing `POST /auth/profile`.

`GET /profile` also returns nullable `phone_number` for the caller. This is contact
information, not a verified login identifier. `PUT /profile` accepts an optional
`phone_number`; omission preserves it, while null and empty values are invalid.
Local `0`-prefixed numbers normalize to `+84`, with spaces/hyphens removed; the
result must be E.164. Invalid format returns `422 invalid_phone`; an unavailable
number returns `409 phone_change_unavailable`. Contact phone changes do not
update Supabase Auth, change the login email, or send SMS/OTP.

The condition-link `{id}` is `user_conditions.id`, not the catalog condition ID.
For PATCH, omitted fields are unchanged and explicit `null` clears the value; an
empty body or a field other than `subtype`/`stage` is invalid. A duplicate POST
returns `409 condition_already_exists`; a missing catalog condition returns
`422 condition_not_found`. Missing or other-user links return the same
`404 user_condition_not_found`. A repeated DELETE returns 404. No idempotency key
or offline queue is introduced: after an uncertain POST outcome, read the profile
and reconcile before retrying.

Health JSON success responses (`GET /profile`, POST/PATCH condition links) carry
`Cache-Control: no-store`. This contract does not promise that header on all
profile errors or on bodyless PUT/DELETE responses. Consult each operation's
response definitions for validation, consent, conflict and provider errors.

## Adherence history

`GET /adherence` retains its current authorization and `Cache-Control: no-store`
on success and errors. Times use `Asia/Ho_Chi_Minh`. Display states are:

- `da_uong` / `uong_tre`: recorded confirmations with a non-null `taken_at`.
- `cho_xac_nhan`: no valid confirmation through exactly `scheduled_at + 120 minutes`.
- `chua_xac_minh`: overdue without a log and without a patient sync checkpoint at
  or after that deadline; it is excluded from the adherence-rate denominator.
- `bo_lo`: recorded missed dose, or an overdue occurrence verified by the patient's
  sync checkpoint. Elapsed time alone does not verify an unlogged missed dose.

Pending/unverified states use `status_source=inferred` and `taken_at=null`.
`patient_last_synced_at` is required but nullable. A premature legacy `bo_lo` row
is displayed as pending through the deadline without rewriting the stored row.

`counted = taken + late + missed`; `total = counted + pending + unverified`.
The rate is `100 * (taken + late) / counted`, rounded half up to two decimal
places, or `null` when `counted=0`. Overall rates use overall counts.
The complete schemas, conditional rules and examples are in `openapi.json`.
