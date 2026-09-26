# Portfolio Manager

[![CI](https://github.com/damianhoward/portfolio-manager/actions/workflows/ci.yml/badge.svg)](https://github.com/damianhoward/portfolio-manager/actions/workflows/ci.yml)
[![CodeQL](https://github.com/damianhoward/portfolio-manager/actions/workflows/codeql.yml/badge.svg)](https://github.com/damianhoward/portfolio-manager/actions/workflows/codeql.yml)
[![codecov](https://codecov.io/gh/damianhoward/portfolio-manager/graph/badge.svg)](https://codecov.io/gh/damianhoward/portfolio-manager)

Two Kotlin **venue-local exchange clients** — `BinanceGateway` and `BitfinexGateway` — that read wallet balances and movement history from **Binance** and **Bitfinex** via their authenticated REST APIs. They share neither a base class nor an interface; what they share is shape and approach, not types. A third exchange would be one new package, modeled on the existing two.

## Gateway design and safety

- **Per-venue gateways** — `BinanceGateway` covers system status / wallets; `BitfinexGateway` covers wallets, movements, settings, and (opt-in) withdrawals. Each is concrete and venue-local — no shared `Gateway` interface.
- **`MessageSender` boundary** — HTTP is behind a class with an injectable `HttpMessageSender` collaborator, so signing logic, request building, and response mapping are unit-testable with no network (see `binance/MessageSender.kt`, `bitfinex/MessageSender.kt`)
- **Withdrawal safety** — Bitfinex's `--withdraw` CLI is dry-run by default; an actual submission requires the explicit `--confirm-withdrawal` flag, and every attempt is appended to a redacted audit log. Mistakes here cost money, so the bar is "you have to mean it".
- **Secret handling** — API key/secret read from env vars only. No creds in code, no creds in committed config, no creds in the JAR
- **Per-venue request signing** — HMAC-SHA256 (Binance) and HMAC-SHA384 (Bitfinex), each with a unit-tested signer

## Architecture

```
   ┌─────────────────────┐                ┌──────────────────┐
   │  BinanceGateway     │───────────────▶│ Binance REST     │
   │   ├─ MessageSender  │  signed HTTP   │ api.binance.com  │
   │   └─ SigningHelper  │                └──────────────────┘
   └─────────────────────┘
   ┌─────────────────────┐                ┌──────────────────┐
   │  BitfinexGateway    │───────────────▶│ Bitfinex REST    │
   │   ├─ MessageSender  │  signed HTTP   │ api.bitfinex.com │
   │   └─ SigningHelper  │                └──────────────────┘
   └─────────────────────┘
```

Each venue lives in its own package — no shared "base client" inheritance, no premature unification. The two gateways look similar but stay independent; if a `Portfolio` aggregation layer is ever added it would compose them rather than generalise them.

## Prerequisites

- JDK 25
- An exchange account with API key + secret on whichever venue you want to hit

### Java compatibility

Java 25 is the supported baseline: Kotlin and Java sources compile to JVM 25 bytecode, so older runtimes are not supported. The checked-in Gradle wrapper pins the build tooling, but it does not install the JDK; install a JDK 25 distribution and make it available through `JAVA_HOME` or `PATH`.

```bash
java -version
./gradlew --version
```

## Setup

```bash
cp .env.example .env
# edit .env, then `source .env` (or use direnv)
```

| Variable              | Used by  |
| --------------------- | -------- |
| `BINANCE_API_KEY`     | Binance  |
| `BINANCE_API_SECRET`  | Binance  |
| `BITFINEX_API_KEY`    | Bitfinex |
| `BITFINEX_API_SECRET` | Bitfinex |

> **Use read-only API keys when exploring.** Withdrawal endpoints need explicit trade/withdraw scopes. Only enable those scopes when you actually intend to call `submitWithdrawalRequest`.

## Running

The Gradle `application` plugin entry point is `binance.MainKt`. It reads system status and
wallet balances, and takes no arguments — the Binance side has no withdrawal path, so it exits
`64` rather than accept arguments it cannot act on:

```bash
./gradlew run
```

For the Bitfinex side, either change `mainClass` in `build.gradle` or run `bitfinex.MainKt` directly. The Bitfinex `Main` defaults to read-only behavior:

1. List wallet balances
2. List recent movements for a currency
3. Print API key permissions

Withdrawal is opt-in **and dry-run by default**. These commands run the Bitfinex CLI, so they
require `mainClass` in `build.gradle` switched to
`com.damianhoward.portfolio.bitfinex.MainKt` first — as-is, `run` boots the Binance side:

```bash
# Dry run (default): logs the redacted intent + journals a DRY_RUN record, no withdraw call.
./gradlew run --args="--withdraw BTC 0.01 destination-address"

# Actually submit. Requires the explicit confirm flag — easy to omit, by design.
./gradlew run --args="--withdraw BTC 0.01 destination-address --confirm-withdrawal"
```

Every withdrawal attempt appends JSON lines to
`~/.portfolio-manager/bitfinex-withdrawals.jsonl` (override with `PORTFOLIO_AUDIT_DIR`), one per
state change, `fsync`ed before the run continues. Destination addresses are redacted in both logs
and the journal (`bc1q***x9` style); the full address is only ever sent to the exchange.

### Withdrawal lifecycle

A withdrawal is `INTENT` → `SUBMITTED` → `CONFIRMED`, and only `CONFIRMED` and `FAILED` are
terminal. The intent is on disk before the venue is contacted, because everything after that
point can fail in a way that leaves money moving.

A timeout is recorded as `UNKNOWN`, never `FAILED`. A dropped connection says nothing about
whether Bitfinex accepted the request, and calling that a failure is what invites a retry into a
second withdrawal. `UNKNOWN` never resolves itself: the next run reconciles it against the venue's
movement history, and until it reaches a terminal state the CLI refuses to submit anything new and
exits `75`.

Reconciliation matches on currency, amount and destination within a time window, because
Bitfinex's `payment_id` is a destination memo rather than a client order id. That is a heuristic
and is treated as one — a single unambiguous match resolves, and two matches, an unrecognised
status or an unreadable movement history all stay unresolved for a human. Absence of a movement
only counts as failure after a settling window, since a movement that has not been published yet
looks exactly like one that never existed.

Exit codes follow the BSD `sysexits.h` convention: `64` on bad arguments
(unknown currency, non-positive amount, blank address), `1` on read or
submission failure, `0` on success.

## Rate limiting and retry

Reads are spaced before the request and retried after a refusal. The two are not interchangeable: a client that only backs off _after_ a 429 has already spent the request that earned it, and on a venue counting refusals toward a ban the cheapest call is the one not made.

Retry is **bounded**, and that bound is the point — an unbounded retry against a rate limiter is how a rate limit becomes a **ban**, which costs the account rather than the request. Five attempts over roughly eight seconds, exponential backoff with full jitter, and a distinct `RetriesExhausted` carrying the attempt count and the last cause. A venue's own wait hint always beats the computed backoff: it knows when it will accept traffic again, and a client guessing shorter earns the next refusal.

`RetryPolicy` is shared between the two venue packages even though the gateways deliberately are not. It is the one thing here that protects an _invariant_ rather than saving typing, which is the test `workspace-config` doc 02 sets for extracting anything.

**Which failure is transient stays venue-local, because the venues disagree.** Binance distinguishes **418 from 429**: 429 means slow down, 418 means an earlier 429 was ignored and the IP is banned for minutes to days. So 418 is _fatal_ here — retrying into a ban extends it. A classifier keyed on status alone, shared across both, would get that wrong.

**Withdrawals are never retried.** A retried read costs a duplicate response; a retried withdrawal risks a duplicate payment, and Bitfinex offers no idempotency key that would make one safe — which is why `WithdrawalReconciler` exists at all. That path stays a single attempt whose uncertain outcome is recorded as `UNKNOWN` and settled against movement history.

## Balance reconciliation

Every Bitfinex read records a **balance snapshot**, and compares it with the previous one against the venue's own movement history. The property asserted is conservation: **a currency's balance change between two snapshots must equal the signed sum of the movements in that window.**

That is worth asserting because the two sides have independent origins. The balance comes from the wallets endpoint and the movements from the movements endpoint; there is no code here that could make them agree by construction, so agreement is evidence rather than tautology. It is the same property `position-ledger` asserts over its fill ledger, in a domain where the write path belongs to someone else entirely.

Balances are summed across wallet types before comparing. Bitfinex splits a currency over `exchange`, `margin` and `funding`, and an internal transfer between them is not a movement — reconciling per wallet type would report every such transfer as unexplained on both sides.

**A verdict is withheld rather than guessed.** Snapshots carry this process's clock and movements carry the venue's, so the two window boundaries do not line up exactly. A movement settling within five minutes of either edge makes that currency _inconclusive_: it may fall inside one side and outside the other, and a difference produced that way is measurement, not drift. That check runs _before_ the comparison, so a coincidental match is not reported as a clean window either.

Snapshots are appended, never overwritten, because the question is not "what is the balance" — the venue answers that on demand — but "what did it do between two points". Each write is `fsync`ed, like the withdrawal journal, since the next run measures from it.

**What this does not claim.** A movement is a transfer in or out of the venue, not a fill. On an account that trades, a balance moves without any movement behind it, so an unexplained delta is expected there and means nothing. The check is a statement about custody — it earns its place on an account used for holding and transferring, where the expected answer is zero and anything else wants a human.

## Tests

```bash
./gradlew test
```

The test suite runs without network access, exchange credentials, or environment variables. It covers:

- Binance and Bitfinex HMAC signing against known payloads
- Gateway request construction and response handling with mocked senders
- Movement-response mapping for both venues
- HTTP sender success and failure paths with mocked clients
- Environment/property lookup behavior
- DTO construction and accessor contracts
- Withdrawal CLI: argument parsing, currency / amount / address validation, address redaction, audit-log row format, and the dry-run vs `--confirm-withdrawal` branch in `bitfinex.MainKt`

## Why this design (for the reader)

- **Why no shared HTTP base?** Each venue has its own quirks — Binance puts the signature in the query string, Bitfinex in a header with a nonce. A shared base ends up as a soup of `if (venue == ...)` branches. One package per venue keeps quirks local.
- **Why is `HttpMessageSender` an injectable constructor default, not an interface?** `MessageSender` takes it as `httpMessageSender: HttpMessageSender = HttpMessageSender()`, so a test swaps in a fake without spinning up a mock HTTP server, but there's no separate interface type — one implementation, no seam to abstract behind.

## Stack

- Kotlin 2.4.20 (JVM target 25), Java 25 toolchain
- Apache HttpClient 5.6
- Jackson 2.22 (BOM-managed)
- Apache Commons Codec / Lang3 / IO
- JUnit Jupiter 6.1, Mockito 5.23, Hamcrest 3
- Gradle 9.6

## Project layout

```
src/main/kotlin/
├── binance/        # Binance gateway, signer, message sender, DTOs
└── bitfinex/       # Bitfinex gateway, signer, message sender, DTOs
src/test/kotlin/
├── binance/        # gateway, mapper, sender, property, and signing tests
├── bitfinex/       # gateway, mapper, sender, property, and signing tests
└── PojoTest.kt     # shared DTO contract tests
```

## License

Apache 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
