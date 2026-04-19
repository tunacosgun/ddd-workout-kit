# ddd-workout-kit

> Reference Domain-Driven Design project in Go. A living example of DDD tactical patterns (aggregates, value objects, repositories) wired into an idiomatic Go service with HTTP + gRPC transports, Firestore + Postgres adapters, and event-driven cross-module communication.

<p align="left">
  <img alt="Go" src="https://img.shields.io/badge/Go-1.22%2B-00ADD8?style=flat-square&logo=go&logoColor=white">
  <img alt="Architecture" src="https://img.shields.io/badge/Architecture-DDD%20%7C%20Modular%20Monolith-blue?style=flat-square">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green?style=flat-square">
  <img alt="Maintained" src="https://img.shields.io/badge/Maintained%20by-Tunasoft%20Yazılım-181717?style=flat-square">
</p>

---

## What this is

`ddd-workout-kit` is Tunasoft Yazılım's living-reference project for applying Domain-Driven Design to Go services. It is the codebase we study (and extend) when modeling complex domains — subscription orders, payment lifecycles, activation-and-device-binding — before we commit to the model in a commercial delivery.

Unlike a blog-post-sized DDD demo, this project is:

- **Complete** — end-to-end: trainer/attendee scheduling, payments, hours, users
- **Multi-module** — cleanly split into bounded contexts
- **Multi-transport** — HTTP + gRPC in the same binary
- **Multi-adapter** — Firestore + MySQL persistence, swappable at the port
- **Tested** — unit, integration, and end-to-end

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the bounded-context map and aggregate model, and [`ROADMAP.md`](./ROADMAP.md) for planned work.

## Why we maintain this

In commercial delivery, the hardest work is **modeling the domain**, not wiring HTTP. Having a reference codebase with real DDD patterns — aggregates that enforce invariants, value objects for primitives, repositories as outbound ports — means we can point at "here's how this should look" during code review, not argue from first principles every time.

We also use it as a testing ground for ideas before they go into client projects: the transactional outbox, event-driven cross-context communication, the command / query split in use cases.

## Bounded contexts

The project splits the domain into four contexts, each in its own module:

| Context       | Responsibility                                         |
|---------------|--------------------------------------------------------|
| **trainer**   | Availability schedule, time-slot invariants            |
| **users**     | Account, hours balance, role (trainer / attendee)      |
| **trainings** | Training aggregate, scheduling rules, lifecycle        |
| **gateway**   | HTTP edge, auth, request routing to internal gRPC      |

Contexts communicate **via events**, not direct calls into each other's internals.

## Layout

```
.
├── internal/
│   ├── trainer/
│   │   ├── domain/            # aggregates, value objects
│   │   ├── app/               # use cases (command / query)
│   │   ├── ports/             # HTTP + gRPC handlers
│   │   └── adapters/          # Firestore / MySQL repositories
│   ├── trainings/
│   ├── users/
│   └── common/
│       ├── auth/              # JWT verification
│       ├── genproto/          # gRPC stubs
│       ├── server/            # HTTP + gRPC bootstrap
│       └── logs/              # slog setup
├── docker/                    # per-service Dockerfiles
├── docker-compose.yml
├── api/                       # OpenAPI + protobuf definitions
└── docs/
    └── adr/                   # Architecture Decision Records
```

## Getting started

### Run the full stack locally

```bash
git clone https://github.com/tunacosgun/ddd-workout-kit.git
cd ddd-workout-kit
docker compose up
```

The stack exposes:
- HTTP API at `http://localhost:3000`
- gRPC internal services on ports `3001` / `3002` / `3003`
- Firestore emulator on `localhost:8787`

### Run tests

```bash
go test ./...
make test-e2e
```

## How we use it

This is our **reference tier** — not a template you drop into a new project. For project bootstrapping, see [`clean-go-starter`](https://github.com/tunacosgun/clean-go-starter).

Concretely, we use this project when:

1. Modeling a new bounded context and wanting a pattern to copy from (see `internal/trainings/domain/training.go` for a canonical aggregate)
2. Onboarding team members to the DDD tactical patterns we use
3. Experimenting with a new technical pattern before introducing it into client code
4. Producing example code for proposals — this is the closest living-reference we have for "how we architect Go services"

## Related Tunasoft projects

- [`clean-go-starter`](https://github.com/tunacosgun/clean-go-starter) — production template we derive from this reference
- [`go-cookbook`](https://github.com/tunacosgun/go-cookbook) — snippet-sized examples of webhook, JWT, OAuth, payment patterns
- [`identity-hub`](https://github.com/tunacosgun/identity-hub) — auth server built on these DDD principles

## Credits

Built on top of [`ThreeDotsLabs/wild-workouts-go-ddd-example`](https://github.com/ThreeDotsLabs/wild-workouts-go-ddd-example), one of the canonical DDD-in-Go references. This fork is maintained by Tunahan Coşgun and Duygu Durmuş at Tunasoft Yazılım and serves as our internal reference codebase for DDD delivery.

## License

MIT — see [`LICENSE`](./LICENSE).

## Contact

- 📧 [info@tunahancosgun.dev](mailto:info@tunahancosgun.dev)
- 🌐 [tunahancosgun.dev](https://tunahancosgun.dev)
- 📅 [Book a consultation](https://cal.com/tunacosgun/intro)
