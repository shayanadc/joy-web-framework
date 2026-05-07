# joy-web-framework
Hexagonal architecture by harmonizing EDA, CQRS, Event Sourcing


# Project Manifest and Architecture Guide

A guide for new developers joining the team. Read this first.

---

## 1. Architectural Philosophy

This codebase is built on **four interlocking patterns**, applied in Go:

- **Hexagonal Architecture (Ports & Adapters).** The application core is isolated from the outside world. Anything external — HTTP, databases, message brokers, notification services, the system clock — sits behind an interface (a "port") and is implemented by an "adapter" outside the core.
- **Domain-Driven Design (DDD).** Business rules live in a dedicated `domain/` layer. Aggregates own the invariants they have the data to enforce. Cross-aggregate rules live in domain services. The model is the asset; everything else is replaceable.
- **CQRS.** Commands (writes) and queries (reads) are separated structurally. Each has its own message types, handlers, and folders. Both are first-class use cases.
- **Event-Driven Architecture (EDA) with Choreography.** Aggregates raise domain events. The command handler persists and publishes them to an event bus. Other parts of the system — policies — react to events independently. There is no central orchestrator; we follow choreography, not the saga pattern.
- **Event Sourcing.** The repository stores the events themselves, not the projected state. Aggregates are rebuilt by replaying their event history.

**Core goals of this structure:**

1. The dependency direction is one-way: `adapters → application → domain`. The domain imports nothing from the application or adapters; the application imports nothing from the adapters.
2. Business logic is independent of HTTP, databases, brokers, and clocks.
3. Adding a new reaction to a business event does not require modifying any existing handler — only adding a new policy and one subscription line.
4. **Nothing skips the bus.** Events flow through the event bus; commands flow through the command bus. Handlers never call other handlers; policies never call handlers.
5. **A deployment is a `main.go` file.** Any subset of use cases can be packaged as its own binary from the same monorepo.

---

## 2. Directory Map & Purpose

```
money-transfer/
├── cmd/
│   ├── monolith/main.go
│   ├── api/main.go
│   ├── query-api/main.go
│   ├── policy-worker/main.go
│   └── <use-case>-service/main.go
├── domain/
│   ├── account/
│   │   ├── account.go
│   │   └── events.go
│   └── transfer/
│       ├── transfer_policy.go
│       ├── working_hours.go
│       └── errors.go
├── application/
│   ├── ports/
│   │   ├── driving/
│   │   │   ├── commands/
│   │   │   └── queries/
│   │   └── driven/
│   ├── commands/
│   ├── queries/
│   └── policies/
├── adapters/
│   ├── driving/
│   │   ├── http/
│   │   ├── event_subscribers/
│   │   └── command_subscribers/
│   └── driven/
│       ├── persistence/
│       ├── eventbus/
│       ├── commandbus/
│       ├── notifier/
│       └── clock/
└── internal/
    └── wiring/
        ├── adapters.go
        └── handlers.go
```

### `cmd/<binary>/`

The composition root for each deployable binary. The single place where concrete adapters are constructed and wired to use cases. No business logic. Multiple `main.go` files allow each deployment to instantiate only the use cases it owns.

### `domain/`

The core. Pure business logic. No I/O, no frameworks, no infrastructure vocabulary. Anything that imports `net/http`, a DB driver, the bus, or `time.Now()` directly does **not** belong here.

- `domain/<aggregate>/` — one folder per aggregate (singular noun: `account`). Holds the aggregate root, value objects, and events the aggregate raises.
- `domain/<concept>/` — cross-aggregate domain logic. Domain services, business policies, the `Clock` interface and similar.

### `application/`

Use cases and ports. Orchestration only — no business decisions.

- `application/ports/driving/{commands,queries}/` — pure DTOs describing what the outside wants.
- `application/ports/driven/` — interface contracts the application needs the outside world to satisfy. Includes `EventBus` and `CommandBus` as separate ports.
- `application/commands/` — command handlers (write side of CQRS).
- `application/queries/` — query handlers (read side of CQRS).
- `application/policies/` — event-triggered use cases.

### `adapters/`

Everything that talks to the outside world. Adapters depend inward.

- `adapters/driving/http/` — HTTP handlers.
- `adapters/driving/event_subscribers/` — wires the event bus to policies.
- `adapters/driving/command_subscribers/` — wires the command bus to command handlers.
- `adapters/driven/persistence/` — repository implementations including the event store.
- `adapters/driven/eventbus/` — event bus implementation, topic routing.
- `adapters/driven/commandbus/` — command bus implementation, point-to-point dispatch.
- `adapters/driven/notifier/` — notification implementations.
- `adapters/driven/clock/` — system clock implementation.

### `internal/wiring/`

Shared composition helpers used by every `main.go`. Builds adapters and individual use cases. Avoids repetition across the multiple binaries — see Section 6.

---

## 3. Naming Conventions

All type names are **PascalCase**. Folder and file names follow Go conventions (lowercase, snake_case for files).

| Role | Pattern | Example |
|---|---|---|
| Command message | `{Action}{Target}Command` | `TransferMoneyCommand` |
| Command handler | `{Action}{Target}CommandHandler` | `TransferMoneyCommandHandler` |
| Query message | `Get{Target}Query` (also `Find` / `List` / `Search`) | `GetBalanceQuery` |
| Query handler | `Get{Target}QueryHandler` | `GetBalanceQueryHandler` |
| Query result | `{Target}View` | `AccountBalanceView` |
| Domain event | `{Subject}{PastTenseVerb}Event`, read as a sentence | `MoneyCreditedEvent` |
| Policy | `When{Event}{Responsibility}` | `WhenMoneyCreditedNotifyReceiver` |
| Domain service / business policy type | Descriptive PascalCase, no fixed suffix | `TransferPolicy`, `WorkingHoursPolicy` |
| Driven port | Role noun, no `I` prefix | `AccountRepository`, `EventBus`, `CommandBus`, `Notifier`, `Clock` |

### Policy naming — the rule in full

Policies are named `When{Event}{Responsibility}`. `{Event}` is the trigger, with the `Event` suffix dropped. `{Responsibility}` is what the policy is responsible for **in business terms** — not the specific command it dispatches.

The responsibility must survive implementation changes. `NotifyReceiver` survives whether the notification is email, SMS, or push. `ApplyFees` survives whether `ChargeFeeCommand` is later split, renamed, or replaced. The short form `When{Event}` is **not allowed** — even when only one policy listens to the event today, tomorrow there will be two.

---

## 4. Separation of Concerns — Where Does It Go?

A decision tree, in priority order. **Stop at the first match.**

### Business invariants and rules

1. **Single aggregate has the data?** → On the aggregate.
2. **Cross-aggregate, or depends on ambient context?** → Domain service in `domain/<concept>/`.
3. **Purely operational (rate limiting, auth)?** → Application or middleware. Rare.

**Anemic aggregates are an anti-pattern.** Don't move rules out for "thinness."

### Time, randomness, ambient inputs

The domain never calls `time.Now()` directly. Declare an interface (`Clock`); adapters fulfill it.

### Database / persistence

Interface in `application/ports/driven/`. Implementation in `adapters/driven/persistence/`. Event sourcing: `Append(events)` writes to the log; `Load(id)` rebuilds via `account.NewFromHistory`.

### HTTP / API routes

Routes registered in each binary's `main.go`. HTTP handlers in `adapters/driving/http/` parse, build a DTO, hand off to the use case. No business logic.

### Event publishing

Aggregates raise events. The command handler calls `PullEvents()` on every aggregate touched, then `Repo.Append(events)` followed by `Bus.Publish(events)`. **Persistence before publication.**

### Reacting to events

A new reaction = a new policy file in `application/policies/` named `When{Event}{Responsibility}` + one new `Subscribe` call in `adapters/driving/event_subscribers/`. Existing code is not modified.

### Dispatching follow-up commands from a policy

Policies dispatch through the command bus, never call handlers directly. Two buses, two semantics:

- **Event bus** — publish/subscribe, fan-out, every subscriber receives a copy.
- **Command bus** — point-to-point, exactly one handler bound per command type.

### Topic routing

The handler always passes all events to `Bus.Publish`. The bus adapter in `adapters/driven/eventbus/` decides per-event topic placement.

### Composition / wiring

Only `cmd/<binary>/main.go` constructs concrete adapters. Shared composition helpers live in `internal/wiring/`. Everywhere else, code talks to interfaces.

---

## 5. Gap Analysis

We have not yet locked down: testing strategy and layout, configuration and environment variables, middleware placement, validation, error handling boundaries, transactional concerns and the outbox pattern, logging/tracing/observability, read-model projections, bounded contexts beyond a single one, event schema versioning, dependency boundary enforcement in CI, static assets/templates if relevant.

These are open questions, not problems. The architecture has a clear seam for each of them.

---

## 6. Deployment Topology — One Codebase, Many Binaries

The architecture supports deploying any subset of use cases as its own binary, while keeping all code in the same monorepo. The mechanism is simple: **each deployable service is a `main.go` file** that wires up only the use cases it owns. The application code does not change.

### The principle

Every use case (command handler, query handler, policy) is a self-contained struct depending only on driven ports. The choice of *which* use cases to instantiate is purely a wiring decision. Five `main.go` files = five deployable binaries from the same codebase. Go's linker tree-shakes each binary so that, for instance, the query-api binary does not contain the `TransferMoneyCommandHandler` code.

### Shared wiring helpers

To avoid duplication across binaries, common wiring lives in `internal/wiring/`:

```go
// internal/wiring/adapters.go
type Adapters struct {
    Repo     driven.AccountRepository
    Events   driven.EventBus
    Commands driven.CommandBus
    Notifier driven.Notifier
    Clock    transfer.Clock
}

func LocalAdapters() Adapters  { /* in-memory, single-process */ }
func RemoteAdapters() Adapters { /* real broker, real DB, env-driven */ }
```

```go
// internal/wiring/handlers.go
func BuildTransferMoney(a Adapters) *cmdHandlers.TransferMoneyCommandHandler { /* ... */ }
func BuildGetBalance(a Adapters)    *queries.GetBalanceQueryHandler          { /* ... */ }
func BuildWhenMoneyCreditedNotifyReceiver(a Adapters) *policies.WhenMoneyCreditedNotifyReceiver { /* ... */ }
```

Each builder takes adapters and returns one fully-wired use case. This is the only "shared" wiring; every `main.go` stays small.

### Per-binary composition roots

| Binary | Contains | Use |
|---|---|---|
| `cmd/monolith` | All use cases, `LocalAdapters` | Local development, small deployments |
| `cmd/api` | Command handlers + HTTP, `RemoteAdapters` | Public write API |
| `cmd/query-api` | Query handlers + HTTP, `RemoteAdapters` | Public read API, scaled independently |
| `cmd/policy-worker` | Policies + event subscribers, no HTTP | Background reactor |
| `cmd/<use-case>-service` | A single use case for isolation/scaling | One handler per pod |

Example — the policy worker contains no HTTP and no command handlers:

```go
// cmd/policy-worker/main.go
func main() {
    a := wiring.RemoteAdapters()
    whenCreditedNotifyReceiver := wiring.BuildWhenMoneyCreditedNotifyReceiver(a)
    eventSubs.RegisterPolicies(a.Events, whenCreditedNotifyReceiver)
    select {} // bus drives the work
}
```

Example — the monolith contains everything:

```go
// cmd/monolith/main.go
func main() {
    a := wiring.LocalAdapters()
    transferMoney             := wiring.BuildTransferMoney(a)
    getBalance                := wiring.BuildGetBalance(a)
    whenCreditedNotifyReceiver:= wiring.BuildWhenMoneyCreditedNotifyReceiver(a)

    cmdSubs.Register(a.Commands, transferMoney)
    eventSubs.RegisterPolicies(a.Events, whenCreditedNotifyReceiver)
    http.Handle("/transfer", &httpadapter.TransferHTTPHandler{Handler: transferMoney})
    http.Handle("/balance",  &httpadapter.BalanceHTTPHandler{Handler:  getBalance})
    http.ListenAndServe(":8080", nil)
}
```

Same code, same use cases. The only difference is which lines `main.go` includes.

### The integration seam

The buses become the integration layer between deployments. With the monolith, the in-memory buses dispatch within a single process. With separated services, the bus adapters become network clients (NATS, RabbitMQ, Kafka — pick one). The application code does not know: `EventBus` and `CommandBus` are still just driven ports. The handler still calls `Bus.Publish(events)`. The policy still calls `Commands.Dispatch(cmd)`. **The seam between in-process and over-the-network is one adapter swap** in `wiring.RemoteAdapters()`.

### Build and deploy

```bash
go build -o bin/api              ./cmd/api
go build -o bin/query-api        ./cmd/query-api
go build -o bin/policy-worker    ./cmd/policy-worker
go build -o bin/monolith         ./cmd/monolith
```

One Dockerfile per binary (or one parameterized Dockerfile). Each deploys, scales, and rolls forward independently.

### The rule, in one sentence

**A deployment is a `main.go` file. The use cases it instantiates are the use cases it owns. The bus is what makes them talk to the use cases owned by other deployments.**

---
---

# Event Storming → Code: A Question-Driven Walkthrough

This is a **discovery method** structured around our architecture. We start at the core (the domain) and work outward, ending at adapters and deployments.

The order follows the dependency rule: `domain ← application ← adapters`, and it follows DDD's discipline that **the model is decided before delivery**. Stable things first (business facts, aggregates, invariants), volatile things last (HTTP, brokers, deployment topology).

---

## Round 1 — What happened?

**Concept behind this round:** Event Storming starts by listing **domain events** — facts in past tense that the business cares about. This is also DDD's **ubiquitous language** in action: the events use the same words the business uses. *Money was credited* is what an operations person would say; it's what the type is named. Names that diverge from business vocabulary are translation layers, and translation layers leak understanding.

Events live in the domain because they are domain artifacts. They carry domain data only — no transport metadata, no IDs the bus would care about. The bus adapter handles routing later.

**Questions:**

1. What facts does the business care about during this operation?
2. Are they emitted by one aggregate or several?
3. What information does each fact carry to be useful to future subscribers and replay?

**Answers for transfer money:**

1. Money left the sender; money arrived at the receiver.
2. Two — each `Account` owns its own fact.
3. Account ID, amount, counterparty, timestamp.

**Manifest rules that apply:**

- Events live in `domain/<aggregate>/events.go`.
- Naming: `{Subject}{PastTenseVerb}Event`, read as a sentence.
- `events.go` is the conventional exception to "one type per file."

**Code generated:**

```go
// domain/account/events.go
package account

import "time"

type DomainEvent interface {
    OccurredAt() time.Time
    AggregateID() string
}

type MoneyDebitedEvent struct {
    AccountID string
    Amount    int64
    To        string
    At        time.Time
}

type MoneyCreditedEvent struct {
    AccountID string
    Amount    int64
    From      string
    At        time.Time
}
```

---

## Round 2 — What enforces the rules tied to a single thing?

**Concept behind this round:** This is the **DDD aggregate** — a consistency boundary. The aggregate root is the single entry point; all changes go through it. Rules the aggregate has *all the data to enforce* belong on the aggregate. Moving them out creates an **anemic domain model**, the canonical anti-pattern where aggregates become data holders and rules drift into procedural code.

Event sourcing layers cleanly onto this: state changes go through `raise`, and the aggregate can be rebuilt from history by `NewFromHistory` — a DDD **factory** that knows how to reconstitute the aggregate without leaking its internals.

**Questions:**

1. What entity is the consistency boundary for each fact?
2. What rules can it enforce by itself?
3. How does it change state — directly, or by raising events?

**Answers:**

1. `Account` owns both events.
2. "Sender's balance must cover the transfer" — the canonical invariant. `Account` has the balance.
3. By raising events. State is a function of past events.

**Manifest rules that apply:**

- Aggregates live in `domain/<aggregate>/`, singular noun.
- One folder per aggregate root.
- If the data is there, the rule is there. **No anemic aggregates.**
- Domain types are pure: no HTTP, no DB, no `time.Now()` directly.

**Code generated:**

```go
// domain/account/account.go
package account

import "errors"

var ErrInsufficientFunds = errors.New("insufficient funds")

type Account struct {
    id      string
    balance int64
    pending []DomainEvent
}

func NewFromHistory(id string, history []DomainEvent) *Account { /* replay */ }

func (a *Account) Debit(amount int64, to string) error {
    if a.balance < amount {
        return ErrInsufficientFunds
    }
    a.raise(MoneyDebitedEvent{a.id, amount, to, /* time */})
    return nil
}

func (a *Account) Credit(amount int64, from string)      { /* raise MoneyCreditedEvent */ }
func (a *Account) PullEvents() []DomainEvent              { /* drain pending */ }
```

The check and the state change are in the same method, on the only object that has the data. This is what DDD means by *"the aggregate owns its invariants."*

---

## Round 3 — What rules span multiple things or need outside context?

**Concept behind this round:** When a rule isn't naturally a responsibility of any single aggregate, it goes in a **DDD domain service**. Stateless, pure, in `domain/`. Eric Evans is precise: *"When a significant process or transformation in the domain is not a natural responsibility of an entity or value object, add an operation to the model as a standalone service."* The key word is **significant** — domain services are for genuine domain operations, not for stitching aggregate calls because the handler felt thin.

When the rule needs ambient context (time, exchange rates, configuration), the domain declares an **interface**, not an implementation. `Clock` is declared in `domain/transfer/`; the adapter layer provides the real one.

**Questions:**

1. Are there rules no single aggregate owns?
2. Do any rules depend on context outside the model — time, rates, config?
3. Should that context be a domain interface?

**Answers:**

1. "Transfers only allowed during working hours."
2. Time of day.
3. Yes — a `Clock` interface in the domain.

**Manifest rules that apply:**

- Cross-aggregate logic in `domain/<concept>/`.
- Domain services are stateless.
- The domain never calls `time.Now()` directly.

**Code generated:**

```go
// domain/transfer/working_hours.go
type Clock interface { Now() time.Time }

type WorkingHoursPolicy struct {
    Clock      Clock
    Open, Close int
}

func (p *WorkingHoursPolicy) IsTransferAllowed() error { /* … */ }
```

```go
// domain/transfer/transfer_policy.go
type TransferPolicy struct {
    WorkingHours *WorkingHoursPolicy
}

func (p *TransferPolicy) Execute(sender, receiver *account.Account, amount int64) error {
    if err := p.WorkingHours.IsTransferAllowed(); err != nil { return err }
    if err := sender.Debit(amount, receiver.ID()); err != nil { return err }
    receiver.Credit(amount, sender.ID())
    return nil
}
```

---

## Round 4 — What does the outside world ask the system to do?

**Concept behind this round:** **Driving ports** — CQRS's separation of intent. Commands and queries are pure data describing what the outside wants. They are *messages*, not handlers — a deliberate DDD-style split between expressing an intent and executing it.

**Questions:**

1. Write or read?
2. What's the minimum data to express the intent?
3. For queries: what's the return shape?

**Answers:** `TransferMoneyCommand`; `GetBalanceQuery` returning `AccountBalanceView`.

**Manifest rules that apply:**

- Driving ports in `application/ports/driving/{commands,queries}/`.
- Pure data — no handlers here.

**Code generated:**

```go
// application/ports/driving/commands/transfer_money_command.go
type TransferMoneyCommand struct {
    FromAccountID string
    ToAccountID   string
    Amount        int64
}
```

```go
// application/ports/driving/queries/get_balance_query.go
type GetBalanceQuery struct {
    AccountID string
}
type AccountBalanceView struct {
    AccountID string
    Balance   int64
}
```

---

## Round 5 — What does the application need from the outside?

**Concept behind this round:** **Driven ports** are the application's wishlist. The use cases declare interfaces; adapters fulfill them. This is dependency inversion. It's also DDD's **repository** pattern in its purest form: to the application, the repository looks like an in-memory collection of aggregates. The fact that it's an event store underneath is hidden.

We declare **two buses** — `EventBus` (fan-out) and `CommandBus` (point-to-point). They are different message-delivery semantics and must be different ports.

**Questions:**

1. What does the application need to persist?
2. How does it publish events?
3. How does it dispatch commands when policies issue them?
4. What other side-effecting capabilities?

**Answers:** repository, event bus, command bus, notifier.

**Manifest rules that apply:**

- Driven ports in `application/ports/driven/`.
- One repository per aggregate root.
- Two buses, two ports.

**Code generated:**

```go
// application/ports/driven/account_repository.go
type AccountRepository interface {
    Load(id string) (*account.Account, error)
    Append(events []account.DomainEvent) error
}
```

```go
// application/ports/driven/event_bus.go
type EventBus interface {
    Publish(events []DomainEvent)
    Subscribe(eventType string, h func(DomainEvent))
}
```

```go
// application/ports/driven/command_bus.go
type Command interface { CommandName() string }

type CommandBus interface {
    Dispatch(cmd Command) error
    Register(commandType string, h func(Command) error)
}
```

---

## Round 6 — Who orchestrates the write?

**Concept behind this round:** The **command handler** is DDD's textbook **application service**. It coordinates; it does not decide. Vaughn Vernon: *"The application service should be only as thick as it needs to coordinate, no thicker."* If you want to add an `if` here that encodes a business rule, that rule belongs upstream — on the aggregate or in a domain service.

**Questions:**

1. What does it load?
2. Who decides the rule — handler or domain?
3. What does it do after the domain has decided?

**Answers:** Both accounts; the domain (`TransferPolicy.Execute`); pull, append, publish.

**Manifest rules that apply:**

- Handlers in `application/commands/`.
- **Mechanical only.** No invariants here.
- `Repo.Append` first, then `Bus.Publish`.
- No filtering — relay every event the domain raised.

**Code generated:**

```go
// application/commands/transfer_money_handler.go
type TransferMoneyCommandHandler struct {
    Repo   driven.AccountRepository
    Bus    driven.EventBus
    Policy *transfer.TransferPolicy
}

func (h *TransferMoneyCommandHandler) Handle(cmd cmds.TransferMoneyCommand) error {
    sender, _   := h.Repo.Load(cmd.FromAccountID)
    receiver, _ := h.Repo.Load(cmd.ToAccountID)
    if err := h.Policy.Execute(sender, receiver, cmd.Amount); err != nil { return err }
    events := append(sender.PullEvents(), receiver.PullEvents()...)
    if err := h.Repo.Append(events); err != nil { return err }
    h.Bus.Publish(events)
    return nil
}
```

---

## Round 7 — Who answers reads?

**Concept behind this round:** Query handlers are the read side of CQRS. They have no side effects, publish nothing, change no state. Keeping them structurally separate means the read path can later evolve independently — different storage, projections, caching — without disturbing the write path.

**Questions:**

1. What does the caller need back?
2. What does the handler fetch to produce that view?

**Answers:** `AccountBalanceView`; the account through the repository.

**Manifest rules that apply:**

- Query handlers in `application/queries/`.
- **No writes. No side effects. No publishing.**

**Code generated:**

```go
// application/queries/get_balance_handler.go
type GetBalanceQueryHandler struct {
    Repo driven.AccountRepository
}
func (h *GetBalanceQueryHandler) Handle(q qries.GetBalanceQuery) (qries.AccountBalanceView, error) {
    a, err := h.Repo.Load(q.AccountID)
    if err != nil { return qries.AccountBalanceView{}, err }
    return qries.AccountBalanceView{AccountID: a.ID(), Balance: a.Balance()}, nil
}
```

---

## Round 8 — Who reacts to what happened?

**Concept behind this round:** **EDA with choreography.** A policy reacts to one event and either calls an outbound port or **dispatches a follow-up command via the command bus**. Policies never call handlers directly. Their names describe their **responsibility**, not the command they currently dispatch — because implementations evolve.

**Questions:**

1. For each event, what should happen because it occurred?
2. What is the **responsibility** of each reaction, in business terms?
3. Does the reaction call an outbound port directly, or dispatch a command?

**Answers for transfer money:**

1. When money is credited, the receiver must hear about it.
2. The responsibility is **notifying the receiver**.
3. Direct outbound port — no follow-up command needed.

**Manifest rules that apply:**

- Policies in `application/policies/`.
- **Naming: `When{Event}{Responsibility}`.** Short form is not allowed.
- Responsibility survives implementation changes — `NotifyReceiver`, not `SendEmail`.
- Policies dispatch through `CommandBus`; never call handlers directly.

**Code generated:**

```go
// application/policies/when_money_credited_notify_receiver.go
type WhenMoneyCreditedNotifyReceiver struct {
    Notifier driven.Notifier
}
func (p *WhenMoneyCreditedNotifyReceiver) Handle(e account.DomainEvent) {
    credited, ok := e.(account.MoneyCreditedEvent)
    if !ok { return }
    _ = p.Notifier.Notify(credited.AccountID,
        fmt.Sprintf("you received %d from %s", credited.Amount, credited.From))
}
```

---

## Round 8b — When a "chain" really is a chain — and even then, choreography handles it

This round shows the architecture under more pressure: a single business operation that produces multiple effects across multiple aggregates, with one of those effects being itself a transactional write somewhere else.

### The example

**Invest money** in an investment account. The business says:

1. Debit the source account.
2. Open an investment for that account.
3. Charge a fee against the investment.

The instinct is to write a chained handler: `InvestMoney` → calls `DebitAccount` → calls `CreateInvestment` → calls `ChargeFee`. **Don't do that.** It puts orchestration in a handler, couples handlers to handlers, and re-implements the saga pattern we explicitly rejected.

The architecture's answer comes from one question: *"If step 2 happened but step 3 didn't, would the business consider step 1 to have happened?"*

- **Debit + open investment** must be atomic: if you debit but don't record the investment, the user lost money for nothing. **One consistency boundary.**
- **Charging the fee** is a separate business operation. It can fail and be retried independently. The investment still happened. **A separate consistency boundary, reached through choreography.**

### The domain — one operation, two events

```go
// domain/investment/events.go
type InvestmentOpenedEvent struct {
    InvestmentID string
    AccountID    string
    Amount       int64
    At           time.Time
}
```

```go
// domain/investment/investment.go
type Investment struct {
    id, accountID string
    amount        int64
    pending       []DomainEvent
}

func New(id, accountID string, amount int64) *Investment {
    inv := &Investment{id: id, accountID: accountID, amount: amount}
    inv.raise(InvestmentOpenedEvent{...})
    return inv
}
```

```go
// domain/investment/investment_service.go
type InvestmentService struct{}

func (s *InvestmentService) Open(acc *account.Account, investmentID string, amount int64) (*Investment, error) {
    if err := acc.Debit(amount, investmentID); err != nil { return nil, err }
    inv := New(investmentID, acc.ID(), amount)
    return inv, nil
}
```

The cross-aggregate orchestration (debit + open investment) is a **domain service** — same shape as `TransferPolicy.Execute`. The handler will know nothing about the sequence.

### The application — one command, two aggregates touched

```go
// application/ports/driving/commands/invest_money_command.go
type InvestMoneyCommand struct {
    AccountID, InvestmentID string
    Amount                  int64
}
```

```go
// application/commands/invest_money_handler.go
type InvestMoneyCommandHandler struct {
    Accounts    driven.AccountRepository
    Investments driven.InvestmentRepository
    Bus         driven.EventBus
    Service     *investment.InvestmentService
}

func (h *InvestMoneyCommandHandler) Handle(cmd cmds.InvestMoneyCommand) error {
    acc, err := h.Accounts.Load(cmd.AccountID)
    if err != nil { return err }

    inv, err := h.Service.Open(acc, cmd.InvestmentID, cmd.Amount)
    if err != nil { return err }

    events := append(acc.PullEvents(), inv.PullEvents()...)
    if err := h.Accounts.Append(events);    err != nil { return err }
    if err := h.Investments.Append(events); err != nil { return err }
    h.Bus.Publish(events)
    return nil
}
```

After this runs, the bus carries `MoneyDebitedEvent` and `InvestmentOpenedEvent`. The handler is still mechanical: load, delegate, persist, publish.

### The chain step — a policy that dispatches a command

The fee charge is **not** part of the same consistency boundary. It's a separate operation that reacts to "an investment was opened." This is exactly the policy → command bus → handler pattern.

```go
// application/policies/when_investment_opened_apply_fees.go
type WhenInvestmentOpenedApplyFees struct {
    Commands driven.CommandBus
}

func (p *WhenInvestmentOpenedApplyFees) Handle(e investment.DomainEvent) {
    opened, ok := e.(investment.InvestmentOpenedEvent)
    if !ok { return }
    _ = p.Commands.Dispatch(cmds.ChargeFeeCommand{
        InvestmentID: opened.InvestmentID,
        Amount:       computeFee(opened.Amount),
    })
}
```

A second policy on the same event handles the receiver notification — a different responsibility, a different file:

```go
// application/policies/when_investment_opened_notify_investor.go
type WhenInvestmentOpenedNotifyInvestor struct {
    Notifier driven.Notifier
}
```

Both policies subscribe to `InvestmentOpenedEvent`. Each has its own responsibility. Each is named for that responsibility, not for the command it dispatches. `ApplyFees` survives if `ChargeFeeCommand` is later renamed to `ApplyManagementChargeCommand`. The event stays general; the policies stay decoupled.

### The wiring

```go
events.Subscribe("InvestmentOpenedEvent", whenInvestmentOpenedNotifyInvestor.Handle)
events.Subscribe("InvestmentOpenedEvent", whenInvestmentOpenedApplyFees.Handle)
commands.Register("ChargeFeeCommand", chargeFeeHandler.Handle)
```

### What this demonstrates

- A "chain of three steps" became **one command, two events, two policies, one follow-up command**. Nothing was orchestrated imperatively.
- The handler stayed mechanical. The domain service made the cross-aggregate decision. Policies made the reaction decisions. The buses integrated everything.
- A failure in `ChargeFeeCommand` doesn't roll back the investment. It can be retried independently — which is exactly the property choreography buys you.
- Adding a fourth step (notify the regulator, update a dashboard, log to an audit trail) means one new policy file and one new `Subscribe` line. Nothing existing changes.

This is the pattern for *every* multi-step business operation in this architecture. There is no other pattern.

---

## Round 9 — How does the outside drive the application?

**Concept behind this round:** **Driving adapters** translate external triggers into use case calls. HTTP, event subscribers, and command subscribers all live here — each of them is a moment where something outside the application core calls in.

**Questions:**

1. What protocols can drive a use case?
2. How do events on the bus reach policies?
3. How do commands on the bus reach handlers?

**Manifest rules that apply:**

- Driving adapters in `adapters/driving/`. Subfolder per role.
- HTTP handlers: parse, build DTO, dispatch. No business logic.
- Bus subscriptions use the message type's full name string.

**Code generated:**

```go
// adapters/driving/http/transfer_handler.go
type TransferHTTPHandler struct {
    Handler *commands.TransferMoneyCommandHandler
}
func (h *TransferHTTPHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // decode JSON → TransferMoneyCommand → h.Handler.Handle(...)
}
```

```go
// adapters/driving/event_subscribers/policy_subscriber.go
func RegisterPolicies(bus driven.EventBus,
    whenCreditedNotifyReceiver *policies.WhenMoneyCreditedNotifyReceiver) {
    bus.Subscribe("MoneyCreditedEvent", whenCreditedNotifyReceiver.Handle)
}
```

```go
// adapters/driving/command_subscribers/command_subscriber.go
func RegisterCommandHandlers(bus driven.CommandBus, transfer *commands.TransferMoneyCommandHandler) {
    bus.Register("TransferMoneyCommand", func(c driven.Command) error {
        return transfer.Handle(c.(cmds.TransferMoneyCommand))
    })
}
```

---

## Round 10 — How does the outside fulfill what the application needs?

**Concept behind this round:** **Driven adapters** are concrete implementations of driven ports. Each is a deployment-time choice. Two buses, two adapters: `eventbus/` for fan-out, `commandbus/` for point-to-point.

**Questions:**

1. Where do events live?
2. How does the event bus deliver?
3. How does the command bus dispatch?
4. What provides time?

**Answers (in-memory):** event-log map by aggregate ID; sync fan-out; sync point-to-point with one handler per command type; real `time.Now()`.

**Manifest rules that apply:**

- Driven adapters in `adapters/driven/`.
- Event sourcing: `Append` writes to the log; `Load` rebuilds via `NewFromHistory`.
- **Topic routing for the event bus lives here**, not in the handler or domain.
- The command bus enforces **one handler per command type** — duplicate registration panics at startup.

**Code generated:**

```go
// adapters/driven/persistence/inmemory_account_repo.go
// adapters/driven/eventbus/inmemory_bus.go
// adapters/driven/commandbus/inmemory_bus.go
// adapters/driven/notifier/console_notifier.go
// adapters/driven/clock/system_clock.go
```

---

## Round 11 — How does it all connect?

**Concept behind this round:** The **composition root** is the only place that knows about every concrete adapter. With the deployment-topology rule from the manifest, *each* binary has its own composition root, and shared composition helpers live in `internal/wiring/` to avoid duplication.

**Questions:**

1. What concrete adapters does this deployment use?
2. Which use cases does this deployment own?
3. How are policies subscribed?
4. How are command handlers registered?
5. How are HTTP routes registered?

**Manifest rules that apply:**

- Composition root in `cmd/<binary>/main.go`.
- Shared builders in `internal/wiring/`.
- No business logic in `main.go`.
- Wiring order: driven adapters → domain services → use cases → driving adapters (subscribers, then HTTP) → start.

**Code generated — monolith:**

```go
// cmd/monolith/main.go
func main() {
    a := wiring.LocalAdapters()

    transferMoney             := wiring.BuildTransferMoney(a)
    getBalance                := wiring.BuildGetBalance(a)
    whenCreditedNotifyReceiver:= wiring.BuildWhenMoneyCreditedNotifyReceiver(a)

    cmdSubs.Register(a.Commands, transferMoney)
    eventSubs.RegisterPolicies(a.Events, whenCreditedNotifyReceiver)
    http.Handle("/transfer", &httpadapter.TransferHTTPHandler{Handler: transferMoney})
    http.Handle("/balance",  &httpadapter.BalanceHTTPHandler{Handler:  getBalance})
    http.ListenAndServe(":8080", nil)
}
```

**Code generated — split deployments:**

```go
// cmd/api/main.go — write side only
func main() {
    a := wiring.RemoteAdapters()
    transferMoney := wiring.BuildTransferMoney(a)
    cmdSubs.Register(a.Commands, transferMoney)
    http.Handle("/transfer", &httpadapter.TransferHTTPHandler{Handler: transferMoney})
    http.ListenAndServe(":8080", nil)
}
```

```go
// cmd/policy-worker/main.go — reactors only
func main() {
    a := wiring.RemoteAdapters()
    whenCreditedNotifyReceiver := wiring.BuildWhenMoneyCreditedNotifyReceiver(a)
    eventSubs.RegisterPolicies(a.Events, whenCreditedNotifyReceiver)
    select {}
}
```

Same use cases, same adapter package. Different `main.go`. Different deployable.

---

## The shape of the method

The order: **events → aggregates → cross-aggregate rules → driving ports → driven ports (with both buses) → command handlers → query handlers → policies (named for responsibility) → chained policies via the command bus → driving adapters → driven adapters → wiring per deployment.**

By the time you reach adapters, every business decision has already been answered and located in the domain. By the time you reach `main.go`, every use case has been built. The only remaining decisions are deployment-shaped: which use cases live in this binary, and what concrete adapters fulfill the ports.

This works for any new feature in the system. The same rounds, in the same order, produce the same kinds of files in the same places, with the same naming — and let you deploy any subset as its own service when it grows up enough to need one.
