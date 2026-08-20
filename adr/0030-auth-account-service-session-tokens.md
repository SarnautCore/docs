# ADR 0030 — Auth service: Argon2id passwords, opaque tokens, NATS admission

**Status**: Accepted (2026-08-20)

## Context

M2 opens with login (ADR 0003), and nothing in the tree implements it. `cmd/auth`
is 79 lines that load config, start telemetry, open infrastructure clients and
serve health probes; it exposes no API. `internal/session` admits any peer whose
`ClientHello` carries a matching protocol version, then grants a spawn — there is
no notion of an account, a character, or a credential anywhere on the connection.
`go.mod` has no password-hashing or token library; `golang.org/x/crypto` appears
only as an indirect requirement pulled in by quic-go and grpc.

So every part of this is a first decision, and the shape of the first decision is
what M3 inherits.

## Decision

### 1. Passwords: Argon2id

`golang.org/x/crypto/argon2` from **`golang.org/x/crypto v0.55.0`**, promoted from
the current indirect requirement (`v0.54.0`) to a direct one and bumped to latest
stable per ADR 0008.

Parameters, fixed and identical for every account:

| Parameter | Value |
|---|---|
| Algorithm | Argon2id (`argon2.IDKey`) |
| Time cost (`t`) | 3 |
| Memory (`m`) | 65536 KiB (64 MiB) |
| Parallelism (`p`) | 4 |
| Salt | 16 bytes from `crypto/rand`, per account, regenerated on every password change |
| Key length | 32 bytes |

Stored in `auth.accounts.password_hash` as a PHC string:

```
$argon2id$v=19$m=65536,t=3,p=4$<salt>$<hash>
```

with both fields in `base64.RawStdEncoding`. `x/crypto/argon2` returns raw bytes
and has no PHC encoder, so encoding and parsing are ours — roughly forty lines in
`internal/auth/credential`. Verification re-derives with the parameters read from
the stored string, never with the compile-time constants, and compares using
`crypto/subtle.ConstantTimeCompare`. Carrying the parameters in the record is what
makes raising the cost later a rehash-on-next-login rather than a flag day.

A login against an unknown email still performs one full Argon2id derivation
against a fixed dummy hash before returning, so response time does not disclose
whether an account exists.

### 2. Tokens: opaque, random, server-side

No signed token. Two kinds, both 32 bytes from `crypto/rand` rendered with
`base64.RawURLEncoding` from `encoding/base64` (43 characters), both prefixed so a
leaked string is identifiable at a glance:

| Token | Prefix | Lifetime | Use |
|---|---|---|---|
| Account session | `sarnaut_as_` | 12 h absolute, no sliding renewal | Bearer credential for the auth HTTP API used by launcher and web |
| Shard ticket | `sarnaut_tk_` | 60 s, single use | Minted for one `(account_id, character_id)` pair; presented by the client to the shard on `EnterZoneRequest` |

State lives in Valkey via **`github.com/redis/go-redis/v9 v9.22.0`** (already a
direct requirement; `infra.Clients.Valkey` already opens it), under keys
`auth:session:<sha256hex>` and `auth:ticket:<sha256hex>` with a TTL equal to the
lifetime. **The plaintext token is never written anywhere** — not to Valkey, not
to PostgreSQL, not to a log. Only its `crypto/sha256` digest is stored, so a
Valkey dump yields no usable credential. Ticket redemption is a single `GETDEL`,
which is atomic, so a replayed ticket finds nothing and is refused.

Lifetimes are short on purpose: the ticket only has to survive the round trip from
"press Enter World" to the QUIC handshake, and 60 seconds is generous for that.
Once the ticket is burned, the QUIC connection itself is the session; the account
session token plays no further part in the game connection.

### 3. Auth ↔ shard transport: NATS request/reply

As ADR 0016 specifies. **`github.com/nats-io/nats.go v1.53.1`**, already a direct
requirement, already connected by `infra.openNATS`. Payloads are protobuf messages
in a new `server/proto/sarnaut/auth/v1/auth.proto` (package `sarnaut.auth.v1`),
compiled by the existing `go generate ./proto` step into `gen/sarnaut/auth/v1`,
per ADR 0017. It sits **outside** the `sarnaut/v1` wire tree deliberately:
[ADR 0027](0027-proto-contract-and-wire-evolution.md)'s `PROTO_LOCK.sha256` and
`sync-proto.ps1` cover `proto/sarnaut/v1/*.proto` only, and these are
server-internal service messages that no client ever receives or needs a copy of.

| Subject | Kind | Request → Reply |
|---|---|---|
| `sarnaut.auth.v1.ticket.redeem` | request/reply | `RedeemTicketRequest{ticket}` → `RedeemTicketResponse{account_id, character_id, character_name, chargen_option_id}` |
| `sarnaut.auth.v1.character.list` | request/reply | `ListCharactersRequest{session_token}` → `ListCharactersResponse{characters[]}` |
| `sarnaut.auth.v1.character.create` | request/reply | `CreateCharacterRequest{session_token, name, chargen_option_id}` → `CreateCharacterResponse{character_id, error_code}` |
| `sarnaut.auth.v1.playlock.renew` | request/reply | `RenewPlayLockRequest{character_id, holder}` → `RenewPlayLockResponse{granted}` |
| `sarnaut.auth.v1.playlock.release` | request/reply | `ReleasePlayLockRequest{character_id, holder}` → empty |
| `sarnaut.auth.v1.session.revoked` | publish, fan-out | `SessionRevoked{account_id, character_id, reason}` — every shard drops matching connections |

Auth subscribes to the request/reply subjects in NATS queue group `auth`, so
adding a second auth instance shares load without duplicate delivery.
`session.revoked` is a plain publish with no queue group, because every shard must
see it.

Shard-side request timeout is 2 seconds with **no retry**. A retried
`ticket.redeem` cannot succeed after the `GETDEL` and would only turn one failed
login into two; the client retries the whole flow instead, starting with a fresh
ticket.

### 4. Account-to-character ownership

- One account owns 0..N characters. A character belongs to exactly one account for
  its lifetime. No transfers, no shared characters, no account merging in M2.
- Identifiers are UUIDv7 minted by auth via `uuid.NewV7` from
  **`github.com/google/uuid v1.6.0`**, promoted from indirect to direct. Time-ordered
  UUIDs keep the primary-key index from fragmenting the way v4 does, and give a
  creation ordering without a second column.
- Ownership facts live in the auth service's PostgreSQL schema (`auth.accounts`,
  `auth.characters`; see ADR 0031). **The shard never reads that schema.** The shard
  learns `account_id` and `character_id` from a `ticket.redeem` reply and from
  nothing else. Auth is the only process with credentials for the `auth` schema.
- **One live session per character.** `ticket.redeem` also takes a play lock in
  Valkey — `SET auth:playing:<character_id> <shard-instance-id> NX EX 60` — and
  refuses redemption if the lock is held by a different holder. The shard renews it
  every 20 seconds over `playlock.renew` and releases it on disconnect. The TTL is
  the point: a shard that dies without releasing frees the character in a minute
  rather than forever, and no operator has to go clear a stuck flag.

### 5. Secret logging rule

**No log record, span attribute, metric label, or error string ever contains a
password, an email address, or a token value.** The only derived forms permitted
are `account_id`, `character_id`, `token_id` (first 16 hex characters of the
token's SHA-256), and `email_domain`.

Mechanism, so the rule is not a habit:

- Secret-bearing values are carried in a distinct type, `secret.Value`, declared in
  `internal/auth/secret`. It implements `slog.LogValuer`, `fmt.Stringer`,
  `encoding.TextMarshaler` and `json.Marshaler`, and every one of them returns
  `[redacted]`. An accidental `%v`, `%s`, `%q`, `slog` attribute or JSON encode is
  therefore already redacted; reaching the real bytes requires calling `.Reveal()`,
  which is greppable.
- Request structs, handler signatures and error-wrapping sites take `secret.Value`,
  never `string`, for passwords, emails and tokens.

Test, in `internal/auth`, run by the existing `go test ./...` CI step:

```
TestAuthLogsCarryNoSecrets
```

It installs `slog.NewJSONHandler(&buf, ...)` as the service logger, then drives the
whole surface with sentinel values — `password: "SENTINEL-PW-2f9c41"`,
`email: "sentinel-2f9c41@example.invalid"` — capturing every token the service
mints along the way: register, login, list characters, create character, mint
ticket, redeem ticket, **login with the wrong password, redeem an expired ticket,
redeem a burned ticket, create a character with a taken name, and send a malformed
request**. It then asserts that neither sentinel nor any captured token appears
anywhere in `buf.Bytes()`, and separately that none appears in `err.Error()` for
any error returned. The failure paths are enumerated deliberately: success paths
rarely leak, and error wrapping is where the input string gets stapled to the
message.

## Alternatives rejected

- **bcrypt** (`golang.org/x/crypto/bcrypt`). One import away and it needs no PHC
  encoder. Rejected for the silent truncation of anything past 72 bytes — which
  turns a long passphrase into a short one with no error — and for a fixed memory
  footprint small enough that GPU attack scales linearly with hardware.
- **`github.com/alexedwards/argon2id v1.0.0`**. It is exactly the PHC wrapper we
  are writing, and it is good. Rejected because it is a thin layer over the same
  `x/crypto` primitive we already ship, and the forty lines it saves are forty
  lines we want to be able to read during an incident.
- **JWT** (`github.com/golang-jwt/jwt/v5 v5.3.1`) **or PASETO**. A self-contained
  signed token earns its keep when the verifier cannot cheaply reach the issuer.
  Ours can and must: the shard has to ask auth which character this connection is
  for and take the play lock, so the round trip happens regardless. Given that, a
  signed token buys nothing and costs the ability to revoke immediately on ban or
  kick — a signed token stays valid until it expires, and shortening the expiry to
  compensate just reintroduces the round trip. It also imports the `alg`-confusion
  and key-distribution failure classes for no gain.
- **gRPC directly from shard to auth**, over the grpc dependency already in
  `go.mod`. It works. Rejected because it makes the shard hold a peer address, a
  health policy and a retry policy per service, and gives no fan-out channel for
  `session.revoked`; NATS provides request/reply and fan-out on the one connection
  `internal/infra` already opens.
- **Letting the shard read `auth.characters` directly** with a read-only role. One
  hop fewer. Rejected because it welds two security domains to one schema, and
  makes any auth schema migration a shard outage.

## Consequences

- `cmd/auth` grows an HTTP API on its own listener, using stdlib `net/http` — no
  framework: `POST /v1/accounts`, `POST /v1/sessions`, `DELETE /v1/sessions`,
  `POST /v1/characters`, `POST /v1/characters/name-checks`
  ([ADR 0032](0032-character-creation.md) §3), `POST /v1/tickets`. It also grows
  the NATS responder.
  TLS for that listener follows ADR 0017's dev self-signed story.
- The client-facing `EnterZoneRequest` gains a ticket field, and
  `session.Server.handle` gains an admission step that refuses connections which do
  not redeem. ADR 0033 records why that check lands at the shard rather than at
  ADR 0016's gateway, and what that costs.
- Valkey stops being optional. `infra.Open` currently skips a client whose endpoint
  is unconfigured; auth and shard must fail to start instead, because sessions,
  tickets and play locks have no fallback.
- `go.mod` gains two direct requirements by promotion, not by download:
  `golang.org/x/crypto v0.55.0` and `github.com/google/uuid v1.6.0`.
- Password reset, email verification, multi-factor and rate limiting are out of M2
  scope and each need their own decision. Nothing here is production-ready, and
  production deployment is a hard stop (ADR 0025) regardless.
