# Architecture

Bounded contexts, aggregates, and tactical-DDD patterns used in `ddd-workout-kit`. This is the reference doc we use when teaching DDD within Tunasoft Yazılım.

## Context map

```
┌─────────────────┐      schedules      ┌─────────────────┐
│    trainer      │◀─────(event)────────│    trainings    │
│   (schedule     │                     │   (lifecycle,   │
│    availability) │─────(event)────────▶│    invariants)  │
└────────┬────────┘                     └────────┬────────┘
         │                                       │
         │ hours-update event                    │ booking event
         ▼                                       ▼
                  ┌─────────────────┐
                  │     users       │
                  │ (balance, role) │
                  └─────────────────┘

                         ▲
                         │ HTTP + gRPC
                         │
                  ┌─────────────────┐
                  │    gateway      │  ← public edge
                  │  (auth, fanout) │
                  └─────────────────┘
```

Each context has a **bounded-context boundary** enforced at the module level. Cross-context communication happens either through the gRPC client (query) or via events published onto the event bus (command side effects).

## Tactical patterns

### Aggregates

An aggregate is a **transactional boundary**. `Training` is the canonical example:

```go
type Training struct {
    uuid     string
    userUUID string
    userName string
    trainingTime time.Time
    notes string
    proposedNewTime *time.Time
    moveProposedBy UserType
    canceled bool
}

func NewTraining(uuid, userUUID, userName string, trainingTime time.Time) (*Training, error) {
    if uuid == ""        { return nil, errors.New("empty training uuid") }
    if userUUID == ""    { return nil, errors.New("empty userUUID") }
    if userName == ""    { return nil, errors.New("empty userName") }
    if trainingTime.Before(time.Now()) {
        return nil, errors.New("training in the past")
    }
    return &Training{...}, nil
}
```

Key properties:
- **Invariants are enforced in the constructor** — there is no way to create a valid-but-broken aggregate
- **Methods mutate the aggregate only through explicit operations** — no `SetFoo(x)` accessors; use business-meaningful methods like `Cancel(by UserType)`
- **Identity is immutable** — the UUID is fixed at creation
- **Aggregates are loaded whole and saved whole** — no partial updates

### Value objects

Primitives are wrapped in value objects that carry meaning:

```go
type HoursBalance struct {
    value int
}

func NewHoursBalance(v int) (HoursBalance, error) {
    if v < 0 { return HoursBalance{}, errors.New("negative hours") }
    return HoursBalance{value: v}, nil
}

func (h HoursBalance) Add(other HoursBalance) HoursBalance { ... }
func (h HoursBalance) CanCover(cost HoursBalance) bool { ... }
```

Benefits:
- Impossible to pass a "hours" integer where "minutes" is expected
- Validation happens once, at construction
- Business operations (`Add`, `CanCover`) are discoverable

### Repository as port

```go
// domain/repository.go — the port
type TrainingRepository interface {
    GetTraining(ctx context.Context, uuid string) (*Training, error)
    AddTraining(ctx context.Context, t *Training) error
    UpdateTraining(ctx context.Context, uuid string, fn func(*Training) (*Training, error)) error
}
```

Notable:
- The `UpdateTraining` signature takes a **mutation function**. The repository handles the `load → mutate → save` cycle atomically, and the business code only writes the mutation.
- Multiple **adapter implementations** exist side-by-side: Firestore (`adapters/trainings_firestore.go`) and MySQL (`adapters/trainings_mysql.go`). They are swappable via DI.

### Command / Query split

Use cases are organized into commands (state-changing) and queries (read-only). Each has its own handler type with a single `Handle` method:

```go
type ScheduleTrainingHandler struct {
    repo domain.TrainingRepository
    userService UserService
}

func (h ScheduleTrainingHandler) Handle(ctx context.Context, cmd ScheduleTraining) error {
    // load, invoke domain logic, save
}
```

Queries return DTOs, never domain entities:

```go
type AvailableHoursReadModel struct {
    Hour time.Time
    Available bool
    TrainerName string
}

type AvailableHoursHandler struct { ... }
func (h AvailableHoursHandler) Handle(ctx context.Context, q AvailableHoursQuery) ([]AvailableHoursReadModel, error) { ... }
```

This keeps commands clean (they never accidentally leak read shapes) and lets us optimize queries with read models.

## Transport layer

Each internal service is a gRPC server. The edge is an HTTP gateway:

```
Client → [HTTP] → gateway → [gRPC] → trainer service
                                   → trainings service
                                   → users service
```

HTTP is public, gRPC is internal-only. Authentication is done at the gateway; internal services trust the gateway's propagated identity token.

## Events

Cross-context communication uses events published to a message bus. In the reference implementation the bus is an in-process channel; in production it maps to Redis Streams or NATS. The pattern is:

```
trainer context: schedules a new availability
    │
    └── emits `AvailabilityAdded` event
            │
            └── trainings context handles:
                    possibly offers the slot to matching users
```

Importantly, the emitter **does not know** who (if anyone) is listening. This is the decoupling that makes bounded contexts real.

## Ports & adapters summary

| Port                       | Adapter options                     |
|----------------------------|-------------------------------------|
| `TrainingRepository`       | Firestore, MySQL                    |
| `HourRepository`           | Firestore, MySQL                    |
| `UserService`              | gRPC client to users service        |
| `TrainerService`           | gRPC client to trainer service      |
| `EventBus`                 | In-memory channel, Redis Streams    |

## Testing strategy

- **Unit tests** for domain logic — no I/O, no mocks, just aggregates and value objects
- **Use case tests** with mocked ports — via `counterfeiter`-generated doubles
- **Integration tests** against the Firestore emulator / MySQL in Docker
- **E2E tests** hitting the HTTP gateway with docker-compose-running stack

## What we changed from upstream

- Added consistent **structured logging** (slog) across all services
- Added **outbox pattern** for cross-context event emission on the MySQL adapter
- Added **OpenTelemetry** traces at port boundaries
- Documented the patterns in this architecture doc for internal onboarding

## References

- [ThreeDots Labs — Go with Domain](https://threedots.tech/go-with-domain/) — the excellent blog series this reference is based on
- [Domain-Driven Design](https://domainlanguage.com/ddd/) — Eric Evans
- [Implementing Domain-Driven Design](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/) — Vaughn Vernon
- Sibling repo: [`clean-go-starter`](https://github.com/tunacosgun/clean-go-starter) — production template we derive from this reference
