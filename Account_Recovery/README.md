# Account Recovery (WIP)

![Account Recovery](./Account_Recovery.png)

An event model for a self-service password recovery flow: an admin configures
the recovery-link TTL, a user submits their email from a forgot-password
screen, the system issues a one-time recovery token, emails the user a link,
and the user lands on the link to set a new password. A secondary chapter
covers what happens when a recovery is requested for an email that doesn't
have an account.

## Actors and lanes

The main timeline has two actor lanes — **Admin** and **User** — plus the
standard **Interaction** lane (UI screens, commands, read models), the
**Swimlane** (events), and a **Spec Lane**. An unlabelled actor row hosts the
internal automation that drives the request through to the email.

## Slices (left to right)

### 1. Configure Account Recovery

The admin opens a config screen and submits `Configure Account Recovery`
(`recovery_link_ttl`, e.g. `15m`), emitting `Account Recovery Configured`.
This populates the **Account Recovery Configuration** read model
(`recovery_link_ttl`), which the recovery processor later consults to compute
token expiry.

### 2. Register Account

A `Register Account` command (`email`, `password`) emits `Account Registered`
(`account_id`, `email`, `password_hash`). This slice is the upstream
dependency that makes an email recoverable — the request processor matches the
recovery email against accounts produced here.

### 3. Forgot Password

The user opens a forgot-password screen (carrying `browser_session`, `email`,
`ip_address`) and submits `Request Account Recovery`. The system emits
`Account Recovery Requested` with a generated `recovery_id`, the user's
`email`, `ip_address`, and a *hashed* `browser_session_hash` (note: the raw
session value never lands on the event stream).

### 4. Recovery Confirmation (UI)

A `Confirmation` read model (`email`, `recovery_id`) backs a follow-up screen
shown to the user immediately after submitting the form, confirming the
request is in flight.

### 5. Account Recovery Requests to Process

A list read model, **Account Recovery Requests to Process**
(`account_id`, `email`, `recovery_id`), joins `Account Recovery Requested`
with `Account Registered` so only requests with a matching registered account
appear. This is what the processor consumes.

### 6. Process Account Recovery Request

The **Account Recovery Processor** automation picks up a pending request,
generates a recovery token, and issues
`Issue Account Recovery Token` (`account_id`, `expires_at`,
`recovery_id`, `recovery_token_hash`). The command emits
`Account Recovery Token Issued`. Only the *hash* of the token is persisted;
the raw token leaves the system once, inside the email.

### 7. Send Account Recovery Email *(informational)*

The processor next issues `Send Account Recovery Email` (`email`,
`recovery_id`, raw `recovery_token`), which emits
`Account Recovery Email Sent`. The companion read model **Email** assembles
the outbound message (`from`, `to`, `subject`, `body`) — the body contains a
link like `http://service.net/recovery/:recovery_id?token=abcde12345`. The
user receives it in their inbox (shown as a sketched screen).

### 8. Account Recovery Request Details

When the user clicks the link, the **Account Recovery Request Details** read
model (`account_id`, `email`, `recovery_id`) hydrates the landing page from
the `recovery_id` in the URL.

### 9. Redeem Account Recovery Token

The user submits a new password from the recovery page. The
`Redeem Account Recovery Token` command carries `account_id`, `recovery_id`,
raw `recovery_token`, the new `password`, plus contextual `at`,
`browser_session`, and `ip_address`. On success it emits `Password Set`
(`account_id`, `password_hash`).

**Specifications** (`Redeem Account Recovery Token`):

- **Redeemed before Expiration** — given `Account Recovery Token Issued`
  (`expires_at=2026-05-05 08:07`), when redeemed at `08:03`, emits
  `Account Recovery Token Redeemed`.
- **Redeemed After Expiration** *(error: "Account Recovery Token Expired")* —
  token issued with `expires_at=08:08`, redemption attempted at `08:12` is
  rejected.
- **Redeemed Once** *(error: "Account Recovery Token Already Redeemed")* —
  if `Account Recovery Token Redeemed` has already been emitted for the
  `recovery_id`, a second redemption is rejected.
- **Account Recovery Token must Match** *(error: "Invalid Account Recovery
  Token")* — the supplied raw token is hashed and compared to
  `recovery_token_hash`; a mismatched token (`abcde1234x` vs the issued
  `fd247` hash) is rejected.

### 10. Token Redeemed (confirmation)

Following `Password Set`, the system emits `Account Recovery Token Redeemed`
(`recovery_id`, plus diagnostic flags `ip_matched` and
`browser_session_matched` derived from comparing the redemption context
against the original request). The user lands on a "password updated" screen.

## Sub-chapter: No Account

A secondary timeline, **No Account**, handles the case where
`Account Recovery Requested` arrives for an email with no matching
`Account Registered`. It is wired off the **Process Account Recovery Request**
slice of the main board.

1. **Reject the request** — the **Account Recovery Processor** issues
   `Reject Account Recovery Request` (`recovery_id`, `reason=no_account`),
   emitting `Account Recovery Request Rejected`.
2. **Queue a notification** — the **Notifications To Send** list read model
   picks up rejected requests with `to` (email) and `type`
   (`reject_account_recovery`).
3. **Send the notification** — the **Notification Processor** automation
   drives `Mark Notification Sent` for each queued notification, emitting a
   `notification_id` completion event.

The intent is that recovery requests against unknown emails still produce a
user-visible response (likely an email saying "no account exists for this
address"), without leaking whether an account exists via the main happy-path
timing.

## Key information

- **Context**: `INTERNAL` across all elements.
- **Token handling**: raw `recovery_token` exists only inside the issuance
  command and the outbound email; the event stream only ever stores
  `recovery_token_hash`. Same pattern for `browser_session` →
  `browser_session_hash` on `Account Recovery Requested`.
- **TTL is data, not code**: the link expiry comes from the admin-configured
  `recovery_link_ttl` read model, so it can change without a code release.
- **Forensic flags**: `Account Recovery Token Redeemed` carries
  `ip_matched` / `browser_session_matched` booleans, capturing whether the
  redemption originated from the same client that requested recovery — useful
  for downstream risk/abuse signals without blocking the redemption itself.
- **Status**: marked WIP. The "Send Account Recovery Email" slices are flagged
  *Informational* (illustrative of an external email-delivery concern rather
  than a fully-specified slice), and the `Event` node in the No Account
  notification flow is still a placeholder name.
