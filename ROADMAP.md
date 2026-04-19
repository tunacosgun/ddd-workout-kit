# Roadmap

Planned enhancements to `ddd-workout-kit` as a living-reference for DDD in Go.

## Near term

### 🚧 Payment context
Add a fifth bounded context — `payments` — demonstrating:
- Idempotent webhook receiver (Stripe + iyzico)
- Payment intent aggregate with lifecycle states
- Outbox-driven `PaymentSucceeded` event consumed by `users` context to credit hours

### 🚧 Sagas for cross-context coordination
A canonical saga pattern for "user books a training, gets charged, gets a confirmation, gets a reminder 24h before." Today the example uses direct event handling; a saga orchestrator makes the flow explicit and testable.

### 🚧 Read-model projection examples
Two projections: live availability view (high read frequency) and reporting view (batch hourly). Demonstrates how to scale reads without touching write-side aggregates.

## Next

### ⏳ ADR series
Convert the design decisions in this project into a series of Architecture Decision Records in `docs/adr/`:

- [ ] ADR-001: Aggregates hold pointers vs values
- [ ] ADR-002: Update-via-mutation-function vs load-mutate-save
- [ ] ADR-003: gRPC for internal transport, HTTP for edge
- [ ] ADR-004: Firestore vs MySQL adapter selection rules
- [ ] ADR-005: Outbox pattern over two-phase commit
- [ ] ADR-006: Events-first cross-context communication

### ⏳ Property-based tests for aggregates
Use `gopter` to generate randomized aggregate operation sequences and verify invariants. A fantastic teaching tool for "what does an invariant actually protect against?"

### ⏳ Bounded-context import linter
A Go analyzer that enforces "module A cannot import from module B's internals." Catches accidental leakage at CI time. Likely packaged separately and consumed by `clean-go-starter`.

## Later

### 📦 OpenTelemetry trace exemplars
Link Prometheus exemplars to OpenTelemetry traces so operators can click from a latency histogram to a concrete slow trace.

### 📦 Alternative transport: NATS
A branch that swaps the in-memory event bus for NATS JetStream. Demonstrates what changes (surprisingly little, if ports are designed well).

### 📦 Multi-tenancy example
Add a tenancy dimension to the aggregate model and show how repositories + queries enforce isolation without pushing the concern into domain code.

## Accepted, not scheduled

- TypeScript SDK generation from the protobuf definitions
- Admin panel (React) for the same domain
- Mobile client (Flutter) as an integration example

## Non-goals

- ❌ Becoming a "drop-in" template — [`clean-go-starter`](https://github.com/tunacosgun/clean-go-starter) is the template; this is a reference
- ❌ Performance benchmarks — the code is optimized for clarity, not throughput
- ❌ Replacing the ThreeDots Labs blog series — we link to it; we extend it

## Release cadence

Tagged when a meaningful new pattern example lands.

## Contributing to the roadmap

Contact [info@tunahancosgun.dev](mailto:info@tunahancosgun.dev) to prioritize an example or pattern for a commercial engagement.
