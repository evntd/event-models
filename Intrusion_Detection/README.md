# Intrusion Detection

An event model for an IP-reputation-based intrusion detection and blocking
system. Authentication failures and probes for suspicious paths each add to a
per-IP reputation score; once an IP's reputation crosses a threshold an
automation blocks it; subsequent requests from that IP are rejected outright;
and a background processor slowly decays reputation over time so that blocks
and offenses age out. The scoring weights, the suspicious-path list, and the
decay schedule are all admin-configurable data, not code.

## Actors and lanes

The timeline uses the standard event-modeling lanes: two configured **actor**
lanes (an **Anonymous** lane, coloured red, for the attacker/unauthenticated
traffic, and a second green actor lane for the operator/admin), an
**Interaction** lane (UI screens, commands, read models), a **Swimlane** for
events, and an empty **Spec Lane** (no specifications have been authored yet).

## Slices (left to right)

### 1. Configure Failed Authentication

An admin screen submits `Configure Failed Authentication`
(`blocked_ip_score=100`, `known_account_score=1`, `unknown_account_score=2`),
emitting `Failed Authentication Configured`. These weights flow into the
**IP Reputation** read model and define how much each kind of offense adds to
an IP's score: a failed login against a real account is cheap (`1`), a failed
login against an unknown account costs more (`2`), and an attempt from an
already-blocked IP is expensive (`100`).

### 2. Configure Suspicious Paths

A `Configure Suspicious Paths` screen submits `Add Suspicious Paths`
(`paths`, a list such as `/wp-admin.php, /phpmyadmin`, and a `score` of `25`),
emitting `Suspicious Path Added`. This seeds the watch-list of honeypot/probe
URLs and the reputation penalty applied when one is requested.

### 3. Failed Login

The **Login** screen drives `Authenticate` (`email`, `ip_address`, `password`,
`user_agent`). A bad password emits `Authentication Failed` (`account_id`,
`ip_address`, `reason=invalid_password`, `user_agent`). The
`ip_address`/`user_agent` are carried as technical attributes. The event feeds
the **IP Reputation** read model, adding `known_account_score` to that IP.

### 4. Failed Login without Account

The same `Authenticate` command for an email with no account
(`not_real@evntd.com`) emits `Authentication Failed` with
`reason=unknown_account`. This is scored separately (and higher) via
`unknown_account_score`, since credential-stuffing against non-existent
accounts is a stronger attack signal than a legitimate user mistyping a
password.

### 5. Suspicious Path Requested

`Request Suspicious Path` (`ip_address`, `path`, `user_agent` — e.g.
`curl/8.11.0` hitting `/phpmyadmin` or `/wp-login.php`) emits
`Suspicious Path Requested`. Any request against a configured suspicious path
adds that path's `score` to the requesting IP's reputation. The slice appears
twice on the board, illustrating different probe paths.

### 6. Update Threat Reputation (IP Blocker)

The **IP Reputation** read model is the heart of the model. It aggregates the
configured scores with every offense to maintain, per IP:

- `ip_address`
- `reputation` (the running score)
- `lifetime_offenses`
- `block_expires_at` / `last_blocked_at`

The **IP Blocker** automation watches this read model (`ip_address`,
`offenses`, `reputation`). When an IP's reputation crosses the block
threshold it issues `Block IP Address` (`ip_address`), emitting
`IP Address Blocked`. That event updates **IP Reputation** (recording
`last_blocked_at` and a `block_expires_at`, bumping `lifetime_offenses`, and
adding the heavy `blocked_ip_score`) and appends the IP to the
**Blocked IP Addresses** list read model (`ip_addresses`).

### 7. Authentication from Blocked IP

When `Authenticate` arrives from an address on the **Blocked IP Addresses**
list, the system short-circuits normal authentication and emits
`Blocked Request Attempted` (`ip_address`, `user_agent`, `path`) instead of
processing credentials. This attempt itself feeds back into **IP Reputation**
(scored at `blocked_ip_score`), so a blocked attacker who keeps trying digs
the hole deeper and pushes their own block expiry further out.

### 8. Configure IP Reputation Decay

A configuration screen submits `Configure IP Reputation Decay`
(`decay_amount=1`, `decay_interval=1h`), emitting
`IP Reputation Configure Decayed`. This populates the
**IP Reputation Decay Configuration** read model (`decay_amount`,
`decay_interval`), the schedule the decay processor consults.

### 9. Periodic Decay Processor / Demonstrate Reputation Decay

The **Reputation Decay Processor** automation reads the decay configuration
and, on each interval, issues `Decay Reputation` (`decay_amount`), emitting
`Reputation Decayed`. Applied to **IP Reputation**, this walks each IP's score
back down over time (e.g. `150 → 149`). The final **IP Reputation** read model
demonstrates the decayed state. Decay is what lets a one-off offender
eventually recover a clean reputation while a persistent attacker — who keeps
re-offending faster than the decay rate — stays blocked.

## Key information

- **Context**: `INTERNAL` across all elements.
- **Reputation is additive, decay is subtractive**: every offense
  (failed login, unknown-account login, suspicious-path probe, blocked-IP
  attempt) adds a configured score to the offending IP; the decay processor
  periodically subtracts a configured amount. An IP stays blocked only while
  offenses outpace decay.
- **Configuration is data, not code**: the scoring weights
  (`Configure Failed Authentication`), the suspicious-path watch-list
  (`Add Suspicious Paths`), and the decay schedule
  (`Configure IP Reputation Decay`) are all admin-driven read models, so they
  can be tuned without a release.
- **Tiered scoring**: `known_account_score=1` < `unknown_account_score=2` ≪
  `blocked_ip_score=100`, plus a suspicious-path `score=25` — distinguishing an
  honest mistake from credential stuffing from an attacker hammering an
  existing block.
- **Single source of truth**: the **IP Reputation** read model is fed by every
  scoring event and is the only input the **IP Blocker** automation needs; the
  derived **Blocked IP Addresses** list is what the authentication path checks.
- **Technical attributes**: `ip_address` and `user_agent` are carried as
  technical attributes on commands/events for forensic context without being
  part of the domain payload.
- **Status**: the **Spec Lane** is empty — no Given/When/Then specifications
  have been written for the scoring, blocking-threshold, or decay rules yet.
