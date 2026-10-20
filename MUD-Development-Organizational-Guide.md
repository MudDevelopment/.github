# MUD Development: An Organizational Guide

**A practical blueprint for planning, building, deploying, and operating a multiplayer text world**

- **Document version:** 1.0
- **Prepared:** October 20, 2026
- **Audience:** Sponsors, product owners, engineering leads, developers, world builders, QA, security, operations, and community teams
- **Reference implementation:** Rust, Tokio, Axum, PostgreSQL, SQLx, and Askama/HTML
- **Status:** Proposed greenfield architecture and delivery guide—not a description of an existing repository or a production-ready implementation

> **Interpretation:** MUD means **Multi-User Dungeon**: a persistent, multiplayer, primarily text-based game. “As an organization” is interpreted as a request for both the technical design and the organizational model needed to deliver and run it. The reference stack reflects a preference for Rust; the operating model and most architectural principles also apply to other languages.

> **Verification boundary:** No existing game source tree was supplied. Paths, interfaces, commands, routes, and configuration in this guide are explicitly proposed. Rust tooling was not present in the authoring environment, so the Rust examples have not been compiled or tested. The guide does not claim measured capacity, current dependency versions, or a validated production deployment.

---

## Contents

1. [Executive recommendation](#1-executive-recommendation)
2. [Game vision and scope](#2-game-vision-and-scope)
3. [Build, adopt, or extend](#3-build-adopt-or-extend)
4. [Organization and decision rights](#4-organization-and-decision-rights)
5. [Delivery process and governance](#5-delivery-process-and-governance)
6. [Reference architecture](#6-reference-architecture)
7. [World model and game systems](#7-world-model-and-game-systems)
8. [Runtime, concurrency, and scheduling](#8-runtime-concurrency-and-scheduling)
9. [Persistence and consistency](#9-persistence-and-consistency)
10. [Player protocols and client experience](#10-player-protocols-and-client-experience)
11. [Repository and coding practices](#11-repository-and-coding-practices)
12. [Rust command parsing example](#12-rust-command-parsing-example)
13. [Content creation and builder tooling](#13-content-creation-and-builder-tooling)
14. [Identity, security, and abuse prevention](#14-identity-security-and-abuse-prevention)
15. [Testing and verification](#15-testing-and-verification)
16. [Configuration and environments](#16-configuration-and-environments)
17. [CI/CD and release management](#17-cicd-and-release-management)
18. [Deployment architecture and procedure](#18-deployment-architecture-and-procedure)
19. [Observability and live operations](#19-observability-and-live-operations)
20. [Backup, recovery, and incident response](#20-backup-recovery-and-incident-response)
21. [Performance and scaling](#21-performance-and-scaling)
22. [Delivery roadmap and estimates](#22-delivery-roadmap-and-estimates)
23. [Organizational templates and checklists](#23-organizational-templates-and-checklists)
24. [Common failure modes](#24-common-failure-modes)
25. [First thirty days](#25-first-thirty-days)
26. [Reference material and glossary](#26-reference-material-and-glossary)

---

## 1. Executive recommendation

Build a **small, coherent, browser-accessible MUD** before building a general-purpose MUD platform.

Start with:

- One world and one authoritative simulation owner.
- A modular monolith rather than independently deployed gameplay microservices.
- A browser client over secure WebSockets.
- HTTPS account management and a separately authorized administration surface.
- PostgreSQL for durable state, with transactions around valuable assets.
- Versioned world content reviewed through the same process as code.
- Automated tests for game rules, ownership, recovery, and abuse cases.
- A named live-service owner and funded community moderation.

Do **not** start with cross-region active-active worlds, unrestricted scripting, player markets, a custom distributed database, or hundreds of unfinished rooms.

### What success means

The first successful release is not “the server accepts connections.” It is a playable loop that survives a restart, resists obvious abuse, and can be supported by the organization.

A useful vertical slice is:

1. Register or log in.
2. Create a character and enter the world.
3. Look, move, communicate, and inspect inventory.
4. Fight one creature and receive a reward.
5. Finish one quest and spend the reward.
6. Disconnect and reconnect without duplicating or losing valuable state.
7. Let a moderator investigate an incident without unrestricted database access.

### Explicitly separate three products

| Product | Users | Primary responsibility |
|---|---|---|
| The game | Players | Enjoyable, fair, accessible interaction |
| The content platform | Builders and designers | Safe creation, validation, preview, publication |
| The live service | Operators and moderators | Availability, recovery, account safety, incident handling |

All three need acceptance criteria. A strong game engine alone does not deliver a maintainable service.

## 2. Game vision and scope

### 2.1 Write a one-page charter

Agree on these decisions before substantial implementation:

| Question | Example starting decision |
|---|---|
| Audience | Adults who enjoy cooperative exploration and text interaction |
| Theme | Original fantasy setting; no unlicensed franchise content |
| Core loop | Explore → encounter → earn → improve → unlock |
| Session length | Useful progress in a short session, with optional longer expeditions |
| Combat | Timed rounds rather than reaction-speed action |
| Social model | Public channels, local speech, groups, direct messages |
| Player conflict | PvE-first; competitive play deferred |
| Death | Recoverable setback with clearly documented consequences |
| Persistence | Characters, inventory, progression, quest state, economy |
| Business model | Decide before adding payment or premium-item systems |
| Accessibility | Keyboard-first, screen-reader-conscious, color optional |
| Launch boundaries | Limited beta, one region, explicit support hours |

These are proposed starting choices, not requirements for every MUD. A roleplay-first world may prioritize emotes and governance over combat; a competitive world requires stronger anti-abuse and balance work earlier.

### 2.2 Define MVP, post-MVP, and exclusions

**MVP**

- Account registration, login, recovery, and logout.
- Character creation with a deliberately small ruleset.
- Rooms, exits, descriptions, objects, and basic NPC behavior.
- Movement, look, help, say, tell, inventory, get, drop, attack, and quit.
- One combat model and one reward path.
- A small connected world with a tutorial and a complete quest.
- Durable character state and transactional item ownership.
- Reconnection with an explicit single-session-per-character policy.
- Moderator mute, kick, suspension, and reviewed restoration tools.
- Metrics, structured logs, backup, restore, and deployment procedures.

**Post-MVP**

- Crafting, guilds, larger quest graphs, richer NPC AI, builder GUI.
- Trading, auctions, mail, player housing, or instanced dungeons.
- Optional traditional-client gateway and negotiated terminal capabilities.
- More sophisticated zone ownership and workload partitioning.

**Explicit exclusions until approved**

- Arbitrary builder-provided native code.
- User-uploaded executable plugins.
- Real-money asset exchange.
- Unlimited procedural world generation.
- Client-authoritative combat or inventory.
- A claim of 24/7 support without a staffed rotation.

### 2.3 Measure both game and service outcomes

Track tutorial completion, return visits, quest completion, abandonment points, report resolution time, and player feedback. Pair these with latency, failed logins, command rejection rates, backup age, and restore-test results.

Do not mistake connection count for engagement. Collect only the analytics needed for stated decisions; publish retention and privacy rules.

## 3. Build, adopt, or extend

A custom engine provides control, but it also creates responsibility for everything an established engine already solves.

| Approach | Strengths | Costs and risks | Best fit |
|---|---|---|---|
| Extend an established MUD engine | Earlier playable world; existing commands and builder tools | Architecture constraints, legacy conventions, licensing review | Small team prioritizing content and community |
| Adopt a modern extensible framework | Good development tooling and existing game abstractions | Language/runtime choice and framework coupling | Team whose desired game fits framework concepts |
| Build a custom Rust engine | Explicit state ownership, type safety, tailored integration | Longest route to mature tools and operations | Engineering-led organization with differentiated requirements |
| Hybrid: existing engine plus web gateway/tools | Preserve gameplay while modernizing access | Two operational stacks and protocol integration | Existing community or valuable legacy world |

Evaluate candidates against real documentation and source, not project names alone. Possible evaluation subjects include Evennia, Ranvier, and established Diku/Circle-derived or other MUD families; inclusion here is not an endorsement of their current maintenance or license terms.

### Decision spike

Time-box an evaluation to produce the same miniature world in each serious candidate. Test:

- Can builders create and publish a room safely?
- Can a player reconnect after a process failure?
- Can inventory updates be made atomic?
- Can permissions be enforced server-side?
- Can the organization operate and patch the stack?
- Are engine, content, assets, and derivative-work licenses compatible with the business model?

Record the result in an architecture decision record. Choose custom development only when the benefits justify the lost head start.

## 4. Organization and decision rights

### 4.1 Organize by responsibilities, not job titles alone

A small team may combine roles. Do not combine all approval powers in the same person simply because headcount is limited.

| Responsibility | Accountable owner | Typical contributors |
|---|---|---|
| Vision, scope, funding | Product owner/sponsor | Design, engineering, community |
| Gameplay rules and balance | Game design lead | Builders, QA, analytics |
| Architecture and code quality | Engineering lead | Backend, client, platform engineers |
| Content quality and publication | World/content lead | Writers, builders, QA |
| Verification and release evidence | QA lead | All engineers and designers |
| Infrastructure and recovery | Service/operations owner | Platform engineer, backend engineers |
| Identity and security review | Security owner | Engineering, operations |
| Conduct, reports, appeals | Community/moderation lead | Moderators, product owner |
| Privacy, licenses, payments | Appropriate legal/business owner | Security, product, finance |

For a small beta, these are often responsibilities spread across several full-time and fractional contributors. A larger public launch needs a separate staffing review. Content and moderation effort should not be hidden inside an engineering estimate.

### 4.2 Decision rights

- Product decides **what outcome** the release serves.
- Design decides **what the rules mean**.
- Engineering decides **how the implementation meets the contract**.
- QA decides whether verification evidence satisfies acceptance criteria.
- Security can block release for unresolved critical exposure under an agreed policy.
- Operations can block release if recovery, capacity, or deployment safety is inadequate.
- The designated release owner records the final go/no-go decision and accepted risks.

A disagreement between a test and implementation must be resolved against the intended rule or contract. Do not silently change tests merely to obtain a green build.

### 4.3 Access separation

Use distinct permissions for player, moderator, builder, content approver, operator, and administrator. A builder should not automatically receive access to player private messages, password hashes, or production infrastructure.

Production actions should use individual identities, least privilege, short-lived elevation where possible, and auditable reasons. Maintain a tightly controlled break-glass path and review every use.

### 4.4 Sustainable operation

Set published support hours and escalation rules. Avoid making a single enthusiastic engineer the permanent on-call team. Document who may stop the economy, disable a zone, or suspend new registrations during an incident.

## 5. Delivery process and governance

### 5.1 Work in playable vertical slices

Prefer “players can complete a persistent quest” over separate months devoted exclusively to database, engine, and client layers. Each slice includes code, content, tests, documentation, deployment, and observability.

A practical cadence:

- Weekly playable demonstration.
- Regular design/content review.
- Short engineering planning and blocker review.
- Release readiness review before each public change.
- Scheduled restore exercise and access review.
- Retrospective after a milestone or significant incident.

### 5.2 Definition of Ready

A work item is ready when it has:

- A player or operator outcome.
- Explicit game rules and edge cases.
- Authorization and persistence requirements.
- Acceptance tests and failure behavior.
- Dependencies and an accountable owner.
- A rough implementation and verification estimate.

### 5.3 Definition of Done

A feature is done when:

- The behavior is implemented and reviewed.
- Automated tests cover intended behavior and meaningful failure cases.
- Content and schema validators pass.
- Authorization, resource bounds, and audit needs are addressed.
- Metrics and actionable errors exist where needed.
- README, DESIGN, relevant help text, and runbooks match the change.
- Migration and rollback implications are documented.
- Staging verification has produced evidence.
- Product/design accepts the delivered player experience.

### 5.4 Separate feature completion from launch readiness

A game can be feature-complete and still unsafe to launch. Launch readiness additionally requires restore evidence, operational ownership, moderation coverage, load testing, privacy documentation, and an understood rollback boundary.

## 6. Reference architecture

### 6.1 Start as a modular monolith

Deploy one application containing well-separated modules. Use library boundaries to preserve separation without adding distributed-system failure modes prematurely.

```text
                       Public network
                             |
                    HTTPS / WSS on 443
                             |
                 TLS reverse proxy / ingress
                             |
              +--------------+---------------+
              |          mud-server          |
              |                              |
              | HTTP account/admin handlers  |
              | WebSocket session adapters   |
              | Authentication/authorization |
              | Command parsing and routing  |
              | Bounded runtime mailboxes    |
              | Authoritative world actor    |
              | Persistence coordination     |
              | Filtered event presentation  |
              +--------------+---------------+
                             |
                    Private database access
                             |
                        PostgreSQL

      Versioned content ----> validation ----> publication
      Private telemetry ----> metrics/logs ----> alerts
      Database backups -----> isolated backup storage
```

### 6.2 Module responsibilities

| Module | Owns | Must not do |
|---|---|---|
| Transport | Connections, framing, timeouts, message limits | Mutate gameplay state directly |
| Session | Authenticated identity, selected character, reconnect policy | Trust client-supplied account roles |
| Command parser | Syntax, aliases, structural validation | Decide whether a player may perform an action |
| Domain rules | Eligibility, damage, movement, rewards, invariants | Depend on HTTP handlers or terminal escapes |
| Runtime | Ordering, ownership, timers, mailbox limits | Perform blocking work in its hot loop |
| Persistence | Transactions, queries, migrations, idempotency | Implicitly decide gameplay rules |
| Presentation | Render authorized events for each recipient | Broadcast hidden information and rely on clients to hide it |
| Content tooling | Schema validation, references, publication | Edit production world state without authorization |
| Administration | Approved privileged workflows and audit | Expose an unrestricted production SQL console |

### 6.3 Command flow

```text
Input frame
  → size/rate check
  → authenticated session context
  → parse typed command
  → authorize action and resolve targets
  → enqueue in authoritative runtime
  → validate current state and determine result
  → commit durable changes when required
  → publish filtered domain events
  → render for each permitted recipient
```

Syntax validation is not authorization. `get sword` can parse correctly while failing because the character is unconscious, the object is owned by someone else, or the room has changed.

### 6.4 Authority model

The server owns health, position, cooldowns, inventory, currency, quest completion, and permissions. Clients submit **intent**, never authoritative state replacements.

Administrative tools also submit typed commands through validated workflows. They do not bypass invariants merely because the caller is an administrator.

## 7. World model and game systems

### 7.1 Fundamental entities

| Entity | Important fields and rules |
|---|---|
| Account | Stable ID, normalized login identity, password hash, status, roles |
| Character | Stable ID, account owner, location, attributes, progression, revision |
| Room definition | Stable content ID, description, exits, tags, encounter references |
| Room instance | Instance ID, definition version, occupants, transient state |
| Exit | Destination, direction/name, visibility, lock and traversal requirements |
| Item definition | Type, description, weight, equipment slot, rule references |
| Item instance | Unique ID, definition reference, owner/container, mutable condition |
| NPC definition | Stats, behavior reference, loot references, spawn rules |
| NPC instance | Current health, location, target, runtime state |
| Quest definition | Stages, requirements, transitions, rewards |
| Quest progress | Character, quest version, completed objectives, reward claim state |
| Encounter | Participants, scheduled actions, deterministic ordering rules |

Distinguish **definition** from **instance**. Two swords may use the same definition but must not share ownership or durability accidentally.

### 7.2 Stable identifiers

Use stable IDs rather than names as keys. Names can change and may collide. Content IDs should survive file moves. Player-facing target resolution should explain ambiguity rather than choose unpredictably.

Example: if two guards are visible, `attack guard` should request disambiguation or use a clearly documented selector such as `attack 2.guard`.

### 7.3 Composition before large inheritance trees

Compose entities from capabilities such as inventory, combatant, merchant, conversational, and container. Do not begin with dozens of deeply nested entity classes.

An entity-component system is optional. Use one only if its complexity pays for itself in measured workloads or clear design flexibility. Ordinary typed structs and explicit rule functions are often enough for the first release.

### 7.4 Movement and visibility

Validate exit existence, destination validity, traversal restrictions, character state, and movement cost in the runtime. Define what happens when a zone is unavailable.

Visibility must be evaluated per viewer. Stealth, private rooms, staff invisibility, and quest-only objects affect not just text rendering but also what targets can be resolved.

### 7.5 Combat

Write the rules before coding:

- How initiative and ties are resolved.
- Whether actions consume time on failure.
- When death, fleeing, and disconnect are evaluated.
- How simultaneous attacks are ordered.
- Who receives experience and loot.
- Whether healing or inventory actions are allowed during combat.
- What reconnecting does to an ongoing encounter.

Inject time and random-number sources into domain logic. This makes tests deterministic. Record enough encounter context to investigate disputes without promising bit-for-bit replay unless you preserve rule/content versions, RNG state, ordering, and all relevant inputs.

### 7.6 Economy

Start with a small number of currency sources and sinks. Treat rewards, purchases, trades, and item transfers as high-integrity operations.

- Represent currency using integer minor units, not floating-point balances.
- Enforce nonnegative balances unless debt is explicitly a feature.
- Give every valuable item exactly one location/owner.
- Make quest reward claims idempotent.
- Audit staff grants and reversals.
- Track issuance and removal; investigate unexplained inflation.

Trading and auctions multiply consistency and fraud problems. Defer them until ownership and recovery are proven.

### 7.7 Quests and narrative

Prefer explicit state machines over ad hoc flags. Keep transitions testable and define version handling for players midway through a quest when content changes.

Avoid trapping a character permanently because a required NPC is missing or a quest item was destroyed. Provide designed recovery routes and audited staff repair tools.

## 8. Runtime, concurrency, and scheduling

### 8.1 Single owner, asynchronous edges

For the MVP, one world actor owns mutable simulation state. Transport tasks communicate with it through bounded mailboxes. This reduces accidental concurrent mutation and makes ordering understandable.

A single actor is **not** a requirement to perform all work on one operating-system thread. Network I/O, password verification, serialization, and database interaction can be delegated; authoritative state transitions remain ordered.

### 8.2 Do not await slow work carelessly

If the world actor waits for every database write, one slow query can stall the entire world. If it launches writes and continues changing the same entities, consistency can break.

Use an explicit pending-operation model:

1. Validate against the current entity revisions.
2. Mark affected entities as reserved for the operation.
3. Send the transaction request to a bounded persistence worker.
4. Continue processing unrelated entities.
5. Receive a completion message carrying the operation ID and result.
6. On success, install committed state and publish the result.
7. On failure, clear the reservation and return a safe error.
8. On ambiguous commit outcome, query by operation ID before retrying.

For a very small first implementation, a short, bounded transaction wait may be acceptable. Measure it, document the limitation, and do not market it as horizontally scalable.

### 8.3 Scheduling model

Use immediate processing for commands such as `look`, and scheduled events for combat rounds, regeneration, effects, and respawns. A timer heap or wheel avoids scanning every entity constantly.

Define:

- Monotonic time for elapsed durations.
- Wall-clock timestamps for audit and durable deadlines.
- Restart behavior for timers and effects.
- A cap on catch-up processing after pauses.
- Deterministic ordering when events share a deadline.

A prototype might use a 100 ms scheduling cadence, but this is an illustrative starting parameter, not a measured requirement.

### 8.4 Bounded resource use

Every queue needs a size limit and full-queue behavior:

| Queue | On overload |
|---|---|
| Incoming player commands | Reject or throttle; do not grow indefinitely |
| Outgoing player events | Drop explicitly replaceable updates or disconnect slow clients |
| Persistence operations | Apply backpressure; stop accepting unsafe mutations |
| Builder publication jobs | Reject or queue within a documented limit |
| Background analytics | Sample/drop according to policy; never block gameplay |

Protect fairness with per-session or per-character command budgets. A single spammer must not consume the entire actor mailbox.

### 8.5 Disconnect and reconnect

Separate account session, socket connection, and character presence. Define a reconnect grace period and a single-controller policy. A second connection must either be rejected or replace the old one through an ordered handoff.

Logging out during combat is a game-design decision, not a transport accident. Persist durable outcomes even if the socket disappears before acknowledgment.

## 9. Persistence and consistency

### 9.1 Classify state by durability need

| State | Recommended treatment |
|---|---|
| Passwords, bans, account ownership | Durable transaction before acknowledging success |
| Currency, item ownership, quest rewards | Durable transaction plus idempotency |
| Important progression | Durable transaction or explicitly documented bounded-loss policy |
| Position outside high-value actions | Periodic checkpoint may be acceptable if designed |
| Combat timers and ephemeral NPCs | Reconstruct or checkpoint according to explicit restart rules |
| Typing indicators and cosmetic output | Transient |
| Moderation actions and staff grants | Durable audit record with controlled access |

“All state is saved every minute” is not a sufficient economy design.

### 9.2 Suggested database areas

```text
accounts                 account credentials and status
account_roles            scoped privileges
sessions                 revocable login sessions
characters               identity, location, progression, revision
item_instances           unique item identity and location
wallets                  currency balances
ledger_entries           economy changes and correlation IDs
quest_progress           versioned progression and reward claim state
operations               idempotency key, result, committed status
moderation_actions       action, actor, subject, reason, timestamps
content_publications     approved content artifact and version
schema_migrations        applied database migrations
```

This is a logical inventory, not a complete SQL schema. Detailed tables need indexes, constraints, retention policies, and migration tests.

### 9.3 Example ownership invariant

The following SQL illustrates a single-location constraint. It assumes referenced character and room-instance tables already exist, and does not by itself enforce capacity, permissions, or prevention of container cycles.

```sql
CREATE TABLE item_instances (
    id UUID PRIMARY KEY,
    definition_id TEXT NOT NULL,
    owner_character_id UUID REFERENCES characters(id),
    room_instance_id UUID REFERENCES room_instances(id),
    container_item_id UUID REFERENCES item_instances(id),
    revision BIGINT NOT NULL DEFAULT 0 CHECK (revision >= 0),
    CHECK (
        num_nonnulls(owner_character_id, room_instance_id, container_item_id) = 1
    ),
    CHECK (container_item_id IS NULL OR container_item_id <> id)
);
```

Cycle prevention for nested containers requires additional validated logic or database enforcement. Choosing to prohibit nested containers in the MVP is simpler.

### 9.4 Atomic purchase procedure

One purchase must not debit money without transferring the item, or transfer the item without debiting money.

```text
Begin transaction
  Authenticate and authorize using server-side identity
  Claim unique (account_id, request_id) with request fingerprint
  If already completed, return the stored result
  If the same ID has different intent, reject it
  Lock participating wallet and item rows in canonical order
  Revalidate funds, price, ownership, capacity, and state revisions
  Debit buyer and credit seller/system ledger
  Transfer item ownership
  Write ledger and audit records
  Store operation result
Commit transaction
Install committed result in authoritative runtime
Emit success to eligible recipients
```

Use consistent lock ordering and bounded retries for deadlock/serialization failures. Do not repeat external side effects inside an automatically retried transaction.

The request ID must be scoped to the authenticated principal and retained for a defined retry window. Never trust a client-provided ID alone as proof that an operation is legitimate.

### 9.5 Crash boundaries

- **Before commit:** nothing valuable should be acknowledged as successful.
- **After commit, before runtime update:** reconcile from the durable operation result.
- **After commit, before client acknowledgment:** a retry returns the same result.
- **During restart:** reconcile durable operations before admitting affected characters.

Use an outbox when committed changes must reliably trigger external notifications. Do not introduce an outbox merely to make ephemeral room narration survive every crash.

### 9.6 Save files, snapshots, and event sourcing

JSON files are useful for content and prototypes; they are a poor default for concurrent account and economy state. A database does not eliminate the need for ownership and transaction design.

Full event sourcing is optional and expensive. It requires event schema evolution, replay compatibility, snapshots, ordering, and operational tooling. Begin with ordinary relational state and focused audit/ledger records unless replay is a core product requirement.

## 10. Player protocols and client experience

### 10.1 Browser-first baseline

Use HTTPS for login, account management, static assets, and operator tools. Use WSS for interactive gameplay.

Proposed application routes:

| Route | Purpose | Access |
|---|---|---|
| `GET /` | Landing/login entry | Public |
| `POST /auth/login` | Establish session | Public, heavily rate-limited and CSRF-protected |
| `POST /auth/logout` | Revoke session | Authenticated, CSRF-protected |
| `GET /play` | Browser game client | Authenticated |
| `GET /ws/game` | WebSocket upgrade | Authenticated, Origin-checked |
| `GET /admin` | Administration UI | Authorized staff |
| `GET /health/live` | Process liveness | Minimal response; exposure controlled |
| `GET /health/ready` | Admission readiness | Primarily infrastructure |
| `GET /metrics` | Telemetry | Private infrastructure only |

These routes are proposed design contracts, not discovered endpoints. Publish real request/response schemas when implementing them.

### 10.2 Versioned message envelopes

Example client command:

```json
{
  "protocol_version": 1,
  "request_id": "b63dd4c6-47e5-47b1-8915-fd234519149d",
  "type": "command",
  "text": "look"
}
```

Example server response:

```json
{
  "protocol_version": 1,
  "type": "room_view",
  "stream_id": "session-stream-82",
  "sequence": 42,
  "request_id": "b63dd4c6-47e5-47b1-8915-fd234519149d",
  "room": {
    "title": "The Lantern Gate",
    "description": "Rain taps against the copper roof.",
    "exits": ["north", "east"]
  }
}
```

Send only information the recipient may see. Internal IDs should be exposed only when needed and must never serve as authorization secrets.

### 10.3 Protocol details to specify

- Maximum frame, message, command, and chat sizes.
- Authentication during upgrade and session expiry afterward.
- Heartbeats, idle timeout, and connection limits.
- Error codes distinct from localized display messages.
- Per-stream sequence meaning and reset behavior.
- Duplicate-command policy and idempotent request handling.
- Server-restart behavior and full-state refresh.
- Backward compatibility and supported client versions.

A sequence number alone does not provide replay. For an MVP, reconnect can issue a new stream ID and an authorized current-state snapshot. If replay is added, define retention, sequence scope, gap detection, and fallback behavior.

### 10.4 Traditional MUD clients

Plain Telnet is not confidential. Do not send reusable passwords across public plaintext connections. If traditional clients are required, evaluate TLS support and compatibility explicitly; document any insecure legacy access as a separate risk decision.

A Telnet gateway must handle IAC negotiation, encoding, line editing, terminal capabilities, ANSI output, and optional extensions such as GMCP if implemented. It is not just “split TCP bytes on newline.”

Both browser and traditional adapters should invoke the same domain commands and permissions.

### 10.5 Accessibility and usability

- Provide a complete keyboard workflow and visible focus.
- Render text safely; do not inject player content as HTML.
- Keep color optional and meaningful without hue alone.
- Make screen-reader announcements configurable to avoid transcript flooding.
- Offer command history, help, predictable errors, and output filtering.
- Support readable typography, contrast, and reduced motion.
- Separate channel output from important combat and system events.
- Test with real assistive technology, not only automated checkers.

## 11. Repository and coding practices

### 11.1 Proposed workspace

```text
mud/
  Cargo.toml
  Cargo.lock
  rust-toolchain.toml
  README.md
  DESIGN.md
  SECURITY.md
  CONTRIBUTING.md
  CODE_OF_CONDUCT.md
  LICENSE
  apps/
    mud-server/
      Cargo.toml
      src/main.rs
    mud-admin-cli/
      Cargo.toml
      src/main.rs
    mud-content-check/
      Cargo.toml
      src/main.rs
  crates/
    mud-domain/
    mud-runtime/
    mud-protocol/
    mud-persistence/
    mud-content/
    mud-test-support/
  web/
    templates/
    static/
  content/
    zones/
    items/
    npcs/
    quests/
  migrations/
  config/
    development.example.toml
  deploy/
    compose/
    systemd/
    proxy/
  docs/
    adr/
    api/
    gameplay/
    runbooks/
  tests/
    integration/
    scenarios/
    load/
```

This is a proposed layout, not a tree already created by this guide. Add crates only when boundaries are useful; avoid splitting every small module into its own package.

Application directory, package, and binary names stay aligned: `apps/mud-server`, package `mud-server`, binary `mud-server`.

### 11.2 Dependency direction

```text
mud-server → runtime, protocol, persistence, content
runtime    → domain and narrow persistence contracts
protocol   → public DTOs and command representations
persistence→ domain identifiers and persistence records
content    → definitions and validation rules
 domain    → minimal foundational dependencies
```

Prevent cyclic dependencies. Domain logic should not import Axum request types or SQL connection pools. Translate at boundaries.

### 11.3 Reference technology choices

| Concern | Starting choice | Rationale |
|---|---|---|
| Language | Rust | Explicit ownership and strong types |
| Async runtime | Tokio | Network tasks, timers, bounded channels |
| HTTP/WebSocket | Axum | Shared service and middleware model |
| Database access | SQLx with PostgreSQL | Explicit SQL and transactional persistence |
| Server-rendered pages | Askama/HTML | Simple account and admin screens |
| Serialization | Serde | Typed content and protocol boundaries |
| Diagnostics | Structured tracing and metrics | Correlated operational evidence |
| Password hashing | Argon2id | Established password-hardening approach |

Choose supported versions during implementation, pin the toolchain and lockfile, and verify actual APIs against those versions. Do not paste unverified version numbers into a production dependency manifest.

### 11.4 Coding standards

- Newtypes for IDs, currency, revisions, and validated names.
- Enums for finite states rather than undocumented strings.
- Explicit `Result` types and stable domain error categories.
- No `unwrap()` on untrusted input or expected runtime failures.
- No blocking filesystem, hashing, or database operations in the simulation loop.
- Bounded channels; no unbounded per-player allocation.
- Parameterized SQL; never construct queries by concatenating player input.
- Inject clock and RNG dependencies for tests.
- Avoid premature `unsafe`; justify and review any necessary use.
- Distinguish user-visible errors from internal diagnostics.

### 11.5 Developer verification loop

Run against the actual workspace once it exists:

```bash
cargo fmt --all -- --check
cargo check --workspace --all-targets --locked
cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
cargo test --workspace --locked
```

If feature combinations are mutually exclusive, replace `--all-features` with a documented CI matrix rather than weakening checks silently. SQLx compile-time query verification needs either a controlled database or properly generated offline metadata; pin the CLI and document the chosen workflow.

Keep local commands in README and deeper rationale in DESIGN. Update both in the same change when behavior or structure changes.

## 12. Rust command parsing example

This dependency-free example illustrates a typed parser boundary. It intentionally supports only `look`, `inventory`, movement, and `say`. It is not a complete game server, transport decoder, Unicode name policy, or authorization system.

The original line is bounded **before** trimming so oversized whitespace cannot bypass the limit. Command matching is ASCII-case-insensitive; player speech retains its case.

```rust
#[derive(Debug, PartialEq, Eq)]
pub enum Direction {
    North,
    South,
    East,
    West,
}

#[derive(Debug, PartialEq, Eq)]
pub enum Command {
    Look,
    Inventory,
    Move(Direction),
    Say(String),
}

#[derive(Debug, PartialEq, Eq)]
pub enum ParseError {
    TooLong,
    UnsupportedControl,
    Empty,
    MissingArgument,
    UnexpectedArgument,
    UnknownCommand,
}

pub fn parse_command(line: &str) -> Result<Command, ParseError> {
    const MAX_COMMAND_BYTES: usize = 512;

    if line.len() > MAX_COMMAND_BYTES {
        return Err(ParseError::TooLong);
    }
    // Transport removes its framing delimiter before calling this function.
    // Reject terminal escapes, embedded newlines, tabs, and other controls.
    if line.chars().any(char::is_control) {
        return Err(ParseError::UnsupportedControl);
    }

    let line = line.trim();
    if line.is_empty() {
        return Err(ParseError::Empty);
    }

    let boundary = line.find(char::is_whitespace).unwrap_or(line.len());
    let verb = line[..boundary].to_ascii_lowercase();
    let argument = line[boundary..].trim();

    if verb == "say" {
        return if argument.is_empty() {
            Err(ParseError::MissingArgument)
        } else {
            Ok(Command::Say(argument.to_owned()))
        };
    }

    let command = match verb.as_str() {
        "look" | "l" => Command::Look,
        "inventory" | "i" => Command::Inventory,
        "north" | "n" => Command::Move(Direction::North),
        "south" | "s" => Command::Move(Direction::South),
        "east" | "e" => Command::Move(Direction::East),
        "west" | "w" => Command::Move(Direction::West),
        _ => return Err(ParseError::UnknownCommand),
    };

    if !argument.is_empty() {
        return Err(ParseError::UnexpectedArgument);
    }
    Ok(command)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn movement_accepts_alias_and_case() {
        assert_eq!(parse_command(" N "), Ok(Command::Move(Direction::North)));
    }

    #[test]
    fn speech_preserves_case() {
        assert_eq!(
            parse_command("SAY Hello, World!"),
            Ok(Command::Say("Hello, World!".to_owned()))
        );
    }

    #[test]
    fn rejects_missing_speech_and_extra_arguments() {
        assert_eq!(parse_command("say"), Err(ParseError::MissingArgument));
        assert_eq!(parse_command("north now"), Err(ParseError::UnexpectedArgument));
    }

    #[test]
    fn rejects_oversized_input_before_trimming() {
        assert_eq!(parse_command(&" ".repeat(513)), Err(ParseError::TooLong));
    }

    #[test]
    fn rejects_terminal_escape() {
        assert_eq!(parse_command("say \u{1b}[2J"), Err(ParseError::UnsupportedControl));
    }

    #[test]
    fn rejects_empty_and_unknown_commands() {
        assert_eq!(parse_command("   "), Err(ParseError::Empty));
        assert_eq!(parse_command("teleport"), Err(ParseError::UnknownCommand));
    }
}
```

### Where the parser belongs

Place it in a domain/protocol module, not directly in a WebSocket callback. After parsing:

1. Attach trusted session and character identifiers.
2. Authorize the requested capability.
3. Resolve targets against the current visible world.
4. Let the authoritative runtime revalidate state.
5. Produce a domain result.
6. Render that result for the recipient.

The use of `unwrap_or` above supplies a safe fallback for an absent whitespace delimiter; it does not panic. The parser's control-character check is not a complete Unicode spoofing or abuse policy. Name normalization and chat moderation remain separate concerns.

**Verification status:** These examples have not been compiled in this environment. Add them to the actual workspace and run the verification loop in Section 11 before adopting them.

## 13. Content creation and builder tooling

### 13.1 Treat content as versioned source

Keep approved definitions in version control. At build/publication time, validate and package them into an immutable artifact identified by a version and digest.

Example room definition:

```json
{
  "schema_version": 1,
  "id": "starter.lantern_gate",
  "title": "The Lantern Gate",
  "description": "Rain taps against the copper roof.",
  "tags": ["safe", "tutorial"],
  "exits": {
    "north": "starter.market_square",
    "east": "starter.training_yard"
  },
  "spawn_table": "starter.gate_encounters"
}
```

The referenced rooms and spawn table must exist before publication. This isolated example is not a complete valid world bundle.

### 13.2 Validation layers

- **Schema:** required fields, supported versions, lengths, numeric bounds.
- **References:** every exit, quest, NPC, item, and spawn reference resolves.
- **Graph:** required locations reachable; intended one-way exits allowed explicitly.
- **Rules:** loot probabilities valid; rewards legal; cooldowns nonnegative.
- **Gameplay:** no unavoidable tutorial dead ends or infinite reward loops.
- **Presentation:** accessible text, content rating, spelling and naming conventions.
- **Security:** output escaped; no arbitrary host paths or unapproved scripts.
- **Compatibility:** active characters and quests have a migration/recovery path.

### 13.3 Publication workflow

```text
Draft → local validation → preview world → peer review
      → gameplay QA → approved immutable bundle → staging
      → authorized production activation → monitoring
```

Never let a production builder edit a shared mutable JSON file without revision checks, review history, or rollback.

### 13.4 Hot reload

Hot reload should be a controlled publication operation:

1. Load and validate the entire candidate bundle separately.
2. Classify compatible and breaking changes.
3. Check entities and quests already using old definitions.
4. Activate at a safe runtime boundary.
5. Retain the prior bundle where old instances still reference it.
6. Audit who activated which digest and when.

Changing descriptions can be easy; removing a room occupied by players is not. Provide fallback destinations and reconciliation logic.

### 13.5 Scripting

Prefer declarative triggers and a small set of approved actions initially. If adding Lua, WASM, or another scripting environment, explicitly bound execution time/fuel, memory, recursion, output, and capabilities. Disable filesystem and network access unless narrowly granted.

A sandbox technology is not automatically a secure policy. Review the host API, not just the interpreter.

## 14. Identity, security, and abuse prevention

### 14.1 Threat model

Assume attackers can create accounts, send arbitrary protocol messages, modify their clients, replay requests, coordinate multiple accounts, and submit malicious text. Also consider compromised staff accounts and supply-chain compromise.

Prioritize account takeover, item duplication, privilege escalation, denial of service, stored XSS, information disclosure, and abuse of other players.

### 14.2 Accounts and sessions

Proposed baseline:

- Browser login form with revocable opaque session cookies.
- Argon2id password hashes with fresh salts and full PHC-format storage.
- Hashing cost benchmarked on deployment hardware.
- Bounded blocking-worker concurrency for password hashing.
- Secure, HttpOnly cookies with an appropriate SameSite policy.
- Session rotation after login or privilege elevation.
- Idle and absolute expiry, plus explicit revocation.
- Strong MFA for staff where supported by the identity design.
- Single-use, short-lived account-recovery tokens stored in a nonrecoverable representation.
- Generic recovery responses to reduce account enumeration.

Do not put password hashes in general game configuration. Do not log passwords, session cookies, recovery tokens, or authorization headers.

Basic Auth is not required for this reference design. Prefer scoped, revocable credentials or organizational identity for staff automation instead of adding a second human login mechanism unnecessarily.

### 14.3 Same-origin is not the same as CSRF protection

Do not enable broad CORS by default. However, omitting CORS headers does not stop all cross-origin requests or replace authorization.

- Validate Origin for browser state-changing requests where applicable.
- Use CSRF protection for cookie-authenticated state changes, including login/logout.
- Check the exact allowed Origin during WebSocket upgrade.
- Define how missing Origin headers are handled for explicitly supported nonbrowser clients.
- Never treat Origin as authentication; nonbrowser clients can forge it.
- Recheck session validity and privilege changes for established sockets.

### 14.4 Server-side permissions

Check authorization for every privileged operation, not merely when rendering the admin menu. Use explicit capabilities such as `content.publish`, `player.mute`, or `economy.adjust`, with scope and reason fields.

Economy adjustments should require stronger review than moving a stuck player. Keep audit storage inaccessible to ordinary moderation tools.

### 14.5 Input and output controls

- Bound decompressed message sizes if transport compression is enabled.
- Limit connection count, login attempts, messages, and expensive commands.
- Escape HTML in every player/content rendering path.
- Strip or encode terminal control sequences for traditional clients.
- Validate Unicode names under a documented normalization/confusable policy.
- Do not execute shell commands from player input.
- Use parameterized database queries and narrowly scoped database roles.

### 14.6 Moderation and privacy

Publish a code of conduct and reporting process before beta. Provide mute, block, report, suspension, and appeal workflows.

Define what staff can inspect, how reports are retained, and when private-message evidence is accessible. Do not log all private conversations “just in case.” Coordinate age restrictions, child-safety obligations, privacy requests, and payment responsibilities with qualified organizational owners for the jurisdictions served.

### 14.7 Supply chain and secrets

Pin dependencies, review licenses, scan for known vulnerabilities and leaked secrets, produce a software bill of materials, and record approved exceptions. Restrict CI credentials and deployment identity.

Mount secrets from an approved secret store or protected runtime files. Example configs contain placeholders only. Backups, logs, and crash dumps need access controls too.

## 15. Testing and verification

### 15.1 Test layers

| Layer | Examples |
|---|---|
| Unit | Parser, damage rules, movement restrictions, quest transitions |
| Property/invariant | No negative balances; each item has one location |
| Integration | Transactions, migrations, session revocation, content activation |
| Protocol | Malformed frames, unknown versions, size limits, reconnect behavior |
| Scenario | Tutorial → combat → reward → purchase → restart |
| Security | Unauthorized actions, CSRF, WebSocket Origin, XSS, rate limits |
| Fault injection | Process kill, database timeout, ambiguous commit, disk-full behavior |
| Load/soak | Busy rooms, slow consumers, command bursts, long sessions |
| Human playtest | Usability, accessibility, fun, balance, griefing opportunities |

### 15.2 Required integrity tests

1. Two players attempt to pick up the same item: exactly one succeeds.
2. A purchase request is retried: it charges once and returns the stored result.
3. A request ID is reused with different intent: it is rejected.
4. The process stops after commit but before acknowledgment: reconnect sees one committed outcome.
5. A quest completion command is repeated: the reward is issued once.
6. A staff account loses permission while connected: later commands are denied.
7. A room is removed in a content update: affected characters receive a safe migration path.
8. The database becomes unavailable: valuable mutations stop safely rather than becoming speculative.
9. A slow client stops reading: queues remain bounded and unrelated players continue.
10. A backup is restored: accounts, inventory, ledgers, and content references reconcile.

### 15.3 Deterministic testing

Use a fake clock and seeded RNG. Record test seed and content version on failure. Avoid sleep-based tests when advancing a simulated clock can prove the rule directly.

Do not change tests or game rules silently to eliminate a failure. Record which contract was authoritative and why.

### 15.4 Load model

Choose an approved beta capacity, then measure realistic mixes:

- Idle connections with heartbeats.
- Active commands per player.
- Chat fan-out in a crowded room.
- Combat across many rooms.
- Login/reconnect storms after restart.
- Content publication during active play.
- Database degradation and backup activity.

For illustration, 200 connected users with 25% issuing two commands per second produces about 100 incoming commands per second. It does **not** predict database load or outgoing traffic; a room-wide broadcast can amplify one command into many deliveries. This is a workload example, not a server capacity claim.

## 16. Configuration and environments

### 16.1 Separate environments

| Environment | Purpose | Data policy |
|---|---|---|
| Local | Fast development | Synthetic accounts and local content |
| CI | Automated verification | Disposable isolated data |
| Staging | Deployment, migration, recovery rehearsal | Synthetic or properly anonymized data |
| Production | Real players | Restricted access and approved retention |

Never let a development default point at production. Use separate credentials, databases, domains, and content-publication channels.

### 16.2 Proposed configuration

```toml
schema_version = 1

[server]
bind_address = "127.0.0.1:8080"
public_origin = "https://mud.example.org"

[game]
world_id = "primary"
content_bundle = "/opt/mud/content/world-v1.json"
max_connections = 200
scheduler_interval_ms = 100

[limits]
max_command_bytes = 512
max_message_bytes = 4096
session_command_queue = 32
session_output_queue = 256

[auth]
session_idle_minutes = 30
session_absolute_hours = 12

[database]
url_secret_file = "/run/secrets/mud_database_url"
max_connections = 20
```

These keys and values are illustrative, not implemented. A loopback bind suits a host-based proxy; a container deployment usually needs an internal interface bind, with isolation enforced by container networking and firewall rules. Queue sizes, timeouts, and pool size require measurement.

### 16.3 Schema evolution

Version runtime config, content schema, database schema, and wire protocol separately. They solve different compatibility problems.

For every change, ask whether an old configuration still loads **and behaves correctly**. Renames, changed encodings, required fields, and changed defaults can be breaking even when parsing succeeds.

Ship migration functions or actionable refusal messages. Reject unknown future versions; do not silently discard unrecognized fields that could represent a typo in a security setting.

Document precedence—for example, defaults → configuration file → approved environment overrides—and log effective nonsecret settings at startup.

## 17. CI/CD and release management

### 17.1 Pull-request gates

1. Formatting, compile checks, linting, and tests.
2. Database migration validation against a clean database and the supported prior version.
3. Content schema/reference validation.
4. Protocol compatibility checks.
5. Dependency, license, and secret scanning.
6. Relevant security and integration tests.
7. Documentation and help-text review.
8. Human review by appropriate code/content owners.

### 17.2 Release artifact

Produce one immutable release record containing:

- Git revision and application version.
- Binary/container digest.
- Toolchain and dependency lockfile identity.
- Content bundle digest and schema version.
- Database migration set.
- Supported client protocol versions.
- Test evidence and known limitations.
- Deployment and rollback instructions.

Build once and promote the same artifact. Do not rebuild “the same tag” separately for production.

### 17.3 Migration discipline

Prefer expand-and-contract changes:

1. Add compatible schema support.
2. Deploy code capable of reading old and new representations.
3. Backfill in bounded batches.
4. Verify correctness and performance.
5. Switch writers/readers deliberately.
6. Remove obsolete representation in a later release.

Run migrations as a single controlled deployment job, not opportunistically from every replica. Estimate lock duration and data-volume effects in staging.

### 17.4 Rollback is not always downgrade

Application rollback may be safe when schema changes remain backward compatible. Destructive migrations may require a forward fix rather than an old binary.

Database restoration can discard valid player actions performed after the restore point. It is an incident-level decision, not a routine deployment rollback. Record the data-loss boundary and player-remediation plan before proceeding.

## 18. Deployment architecture and procedure

### 18.1 Initial production topology

A reasonable beta topology is a TLS reverse proxy, one active world application, PostgreSQL, private telemetry, and isolated backup storage.

Choose managed PostgreSQL when operational simplicity justifies the cost. A single virtual machine can support a small beta but is a single failure domain; a single application with a highly available database still has application downtime during replacement.

Docker Compose can simplify a small deployment, but it does not provide high availability. Kubernetes is optional and does not solve authoritative world ownership by itself.

### 18.2 Proposed network exposure

| Service | Example port | Exposure |
|---|---:|---|
| HTTPS/WSS edge | 443 | Public |
| HTTP redirect/ACME if used | 80 | Public only when required |
| Application HTTP | 8080 | Loopback or private network |
| PostgreSQL | 5432 | Private, application/operator access only |
| Metrics listener, if separate | 9090 | Private telemetry network |
| SSH or management plane | Environment-specific | Restricted organizational access |

Ports are proposed conventions, not known properties of an existing application. Update README and firewall documentation whenever the implementation changes them.

### 18.3 Host and container hardening

- Run as an unprivileged identity.
- Use a read-only application filesystem where practical.
- Mount only required content and secret paths.
- Apply CPU/memory limits with tested failure behavior.
- Drop unnecessary capabilities and avoid host networking unless justified.
- Restrict inbound and outbound traffic.
- Patch base images and hosts under a defined cadence.
- Synchronize system clocks; validate certificate renewal.
- Keep database volumes persistent and out of disposable container layers.

### 18.4 Startup sequence

```text
Read and validate configuration
  → load secrets
  → verify compatible database schema
  → load and validate approved content bundle
  → acquire exclusive world ownership
  → reconcile durable state and pending operations
  → initialize runtime and scheduled events
  → start handlers in not-ready state
  → complete self-checks
  → advertise readiness and admit players
```

Liveness means the process is functioning; readiness means it can safely admit traffic. Do not make liveness fail on every transient database problem and create restart storms.

### 18.5 Safe release runbook

1. Confirm approval, artifact digests, backups, and migration compatibility.
2. Notify players if interruption is expected.
3. Disable new logins and mark the old instance not ready.
4. Pause new state-changing actions at a defined boundary.
5. Finish or reconcile in-flight durable operations.
6. Checkpoint recoverable runtime state and stop the old world owner.
7. Apply approved migrations once.
8. Start the new instance with the approved content bundle.
9. Verify exclusive ownership and startup reconciliation.
10. Smoke-test login, movement, inventory, a reward, and moderation access.
11. Re-enable admission and monitor defined rollback triggers.
12. Record actual release outcome and exceptions.

A reverse proxy cannot drain a long-lived WebSocket merely by ceasing new routing. The server must notify existing clients, enforce a drain deadline, and close connections predictably.

### 18.6 Exclusive world ownership

Never run two writable copies of the same world against the same state without a designed coordination protocol.

A database lock or lease can help coordinate the MVP, but production failover needs a stale-owner story. On loss of the ownership connection, stop mutations and fence the old process before promoting a replacement. For stronger guarantees, attach a monotonically increasing ownership epoch to writes and reject stale epochs atomically.

Until fencing and recovery are tested, prefer supervised manual failover to unsafe automatic promotion.

### 18.7 Graceful shutdown

Handle termination signals by stopping admission, quiescing mutations, waiting for bounded completion, flushing checkpoints, releasing ownership, and exiting. Set the supervisor termination deadline longer than the tested drain window. On deadline expiry, rely on the same crash-recovery path exercised in tests.

## 19. Observability and live operations

### 19.1 Logs

Use structured logs with timestamp, severity, service version, world/zone, correlation ID, operation ID, and safe entity identifiers. Include content version when diagnosing gameplay behavior.

Do not use raw chat or credentials as default log fields. Avoid high-cardinality player identifiers as metric labels; logs and controlled traces can carry identifiers when justified.

### 19.2 Metrics

| Category | Useful signals |
|---|---|
| Connections | Active sessions, reconnects, authentication failures |
| Commands | Rate, rejection reason, queue wait, completion latency |
| Runtime | Scheduler lag, mailbox depth, pending persistence operations |
| Output | Fan-out volume, slow consumers, dropped replaceable events |
| Database | Pool wait, transaction duration, failures, retry count |
| Economy | Currency issued/removed, duplicate-request hits, invariant violations |
| Content | Publication failures, unresolved references, fallback relocations |
| Operations | Backup age, restore-test age, disk use, certificate expiry |
| Community | Report backlog and resolution time without exposing private content |

### 19.3 Proposed service objectives

The following are planning targets to approve and test, not observed results:

| Objective | Initial proposal |
|---|---|
| Availability | 99.5% monthly successful synthetic login-and-look checks, counting maintenance |
| Immediate command latency | p95 server-side completion below 250 ms under approved beta load |
| Integrity | No acknowledged duplicate quest rewards or duplicate item ownership |
| RPO | No more than 5 minutes of durable data loss in a disaster, if backup/WAL design proves it |
| RTO | Restore service within 60 minutes in the tested disaster scenario |

Define measurement scope and exclusions. Network latency is not included in the server-side command target. Recovery targets require drills; storing a backup file does not prove either objective.

### 19.4 Alerts and runbooks

Page for imminent data loss, world unavailability, ownership conflicts, and persistent transaction failures. Use lower-urgency channels for gradual capacity trends and routine report backlog.

Every actionable alert should name the impact, first diagnostic steps, mitigation, and escalation owner. Avoid alerts that merely state “CPU high” without a service consequence.

## 20. Backup, recovery, and incident response

### 20.1 Backup plan

- Use consistent PostgreSQL backups and WAL archiving/PITR where required by the RPO.
- Back up approved content bundles and release manifests.
- Preserve required secrets/encryption keys through a separate controlled recovery process.
- Encrypt backups and keep a copy outside the primary failure domain.
- Apply retention limits and deletion policies.
- Monitor successful completion, integrity checks, and recovery usefulness.

A replica is not a backup: accidental deletion or malicious changes can replicate immediately.

### 20.2 Restore drill

1. Select a recovery point and identify the matching application/content versions.
2. Restore into an isolated environment.
3. Keep email, payment, webhook, and other external side effects disabled.
4. Validate schema compatibility and ownership state.
5. Reconcile accounts, inventory, ledger entries, and quest progress.
6. Run representative player scenarios.
7. Measure actual data loss and elapsed recovery time.
8. Record failures, assign owners, and repeat after correction.

### 20.3 Incident levels

| Severity | Example | Initial response |
|---|---|---|
| Critical | Active duplication, credential exposure, total outage | Incident commander; contain; preserve evidence; communicate |
| High | One zone unavailable, repeated lost sessions, severe moderation incident | Assign owner; mitigate; inform affected players |
| Normal | Isolated defect with workaround | Triage and schedule correction |

### 20.4 Economy duplication incident

- Disable the affected feature or economy mutations with a tested control.
- Preserve operation and ledger evidence.
- Identify the exploit window and affected accounts/assets.
- Fix the invariant, not just the visible symptom.
- Reconcile with reviewed, idempotent repair tooling.
- Test the repair on a restored copy first.
- Communicate player impact without disclosing exploitable detail prematurely.

Never mass-edit production balances from an unreviewed spreadsheet or shell script.

### 20.5 Post-incident review

Document timeline, impact, contributing conditions, detection gaps, recovery evidence, and corrective actions. Use a blameless review while maintaining clear accountability for follow-through.

## 21. Performance and scaling

### 21.1 Measure before partitioning

Profile CPU, queueing, allocation, database wait, serialization, and broadcast fan-out. Optimize the demonstrated bottleneck, not an assumed one.

Common early improvements:

- Cache immutable content definitions.
- Avoid cloning full world state for each command.
- Index high-frequency queries.
- Batch only operations whose durability contracts allow batching.
- Coalesce replaceable status updates.
- Bound chat fan-out and output buffers.
- Separate expensive analytics from gameplay.

### 21.2 Scaling stages

| Stage | Design | New responsibility |
|---|---|---|
| 1 | One application, one world owner | Clear crash recovery and tested capacity |
| 2 | Scale stateless HTTP/edge components | Shared sessions and correct world routing |
| 3 | Multiple zone actors in one process | Cross-zone message ordering |
| 4 | Zones across processes | Ownership leases, fencing, transfer recovery |
| 5 | Multiple independent realms | Realm-aware identity, content, and operations |

Do not put two world processes behind a round-robin load balancer and assume they share coherent simulation state.

### 21.3 Cross-zone transfer

Once zones have separate owners, movement becomes a transaction-like protocol:

1. Source records a uniquely identified transfer intent.
2. Destination reserves admission.
3. Durable ownership changes under a revision/fencing check.
4. Destination activates the character exactly once.
5. Source removes its active copy.
6. Recovery resolves incomplete transfers by durable ownership, not by guessing.

Use an explicit handoff state machine and idempotent messages. Avoid a casual “send character JSON to another server” design.

### 21.4 Multi-region caution

A single shared world across regions introduces latency and consistency tradeoffs into combat and ownership. Independent regional realms are often simpler. Cross-realm transfers still need careful asset and identity handling.

## 22. Delivery roadmap and estimates

The schedule below is an **illustrative planning range**, not a commitment. It assumes a small experienced team, narrow MVP, no legacy migration, no real-money economy, and steady access to product/design decisions. Validate estimates after the vertical slice.

| Phase | Illustrative effort window | Main output | Exit evidence |
|---|---|---|---|
| Discovery and engine decision | 1–2 weeks | Charter, engine spike, risk register | Approved scope and build/adopt ADR |
| Foundation | 2–3 weeks | Workspace, CI, account/session skeleton, initial content | Build and deployment rehearsal |
| Playable vertical slice | 3–5 weeks | Rooms, commands, combat, one quest, persistence | End-to-end play and restart tests |
| Authoring and integrity | 3–5 weeks | Builder validation, inventory/economy transactions, moderation | Duplicate/race/security tests |
| Hardening and closed beta | 3–5 weeks | Load evidence, restore drills, usability fixes | Beta readiness review |
| Launch preparation | 1–2 weeks | Release package, staffing, support and rollback plan | Recorded go/no-go decision |

Sequentially this is roughly 13–22 weeks for the narrowly defined beta path. Content production overlaps but can still become the critical path. Public-service maturity, richer gameplay, and larger content volume may take substantially longer.

### Budget categories

- Engineering, game design, world writing/building, QA.
- Community moderation and support.
- Infrastructure, database, backups, telemetry, domains, certificates.
- Security review, accessibility testing, legal/license review.
- Tooling, CI minutes, asset production, contingency.
- Ongoing maintenance after launch.

Use provider quotes and actual staffing costs for a monetary budget. Do not infer a production cost from an inexpensive prototype server alone.

### Milestone decision rules

- If the engine spike cannot support the integrity model, revisit engine choice.
- If the vertical slice is not enjoyable, revise the loop before increasing world size.
- If recovery cannot meet the target, reduce launch promises or improve the design.
- If moderation cannot cover the proposed launch, restrict access and publish support boundaries.

## 23. Organizational templates and checklists

### 23.1 Architecture decision record

```markdown
# ADR-0001: Single authoritative world owner

Status: Proposed
Owner: Engineering lead
Date: YYYY-MM-DD

## Context
What problem exists? What constraints matter?

## Options
What alternatives were evaluated against actual evidence?

## Decision
What are we choosing, and where does it apply?

## Consequences
Benefits, costs, failure modes, and operational responsibilities.

## Verification
Tests or experiments that support the decision.

## Revisit trigger
A measurable threshold or requirement that warrants reconsideration.
```

### 23.2 Feature issue

```markdown
# Feature: Complete the gatekeeper's first quest

Player outcome:
A new character learns movement, inventory, and quest completion.

Rules:
- Completion requires the designated item.
- The reward is granted once per character.
- Repeating the command returns the prior completion status.

Failure cases:
- Item moved before completion.
- Disconnect after commit, before acknowledgment.
- Quest content changes while progress is active.

Acceptance evidence:
- Automated state-transition and duplicate-request tests.
- Restart test.
- Content review and playtest.

Operations/docs:
- Reward metrics and audited repair path.
- Updated help text and gameplay documentation.
```

### 23.3 Risk register

| Risk | Leading indicator | Owner | Mitigation |
|---|---|---|---|
| Scope expansion | Vertical slice keeps slipping | Product | Fixed MVP and explicit deferred list |
| Item duplication | Ownership/ledger inconsistencies | Engineering | Transactions, idempotency, fault tests |
| Content bottleneck | Engine ready, zones unfinished | Content lead | Early builder pipeline and small world |
| Staff burnout | One person handles all incidents | Service owner | Rotation, support hours, access limits |
| Account compromise | Suspicious logins/elevations | Security | Staff MFA, revocation, audit |
| Unrecoverable backups | Restore drill absent or failing | Operations | Scheduled isolated restore exercises |
| Toxic community | Report backlog and repeat abuse | Community lead | Clear policies, tooling, staffing |
| License conflict | Untracked third-party content | Business/legal | Asset register and license review |

### 23.4 Launch checklist

**Product and community**

- [ ] Core loop accepted through playtesting.
- [ ] Tutorial, help, and accessibility review complete.
- [ ] Conduct, privacy, reporting, and appeal processes published.
- [ ] Support coverage and moderation ownership confirmed.

**Engineering and integrity**

- [ ] Reproducible artifact and pinned dependencies recorded.
- [ ] Protocol/schema compatibility reviewed.
- [ ] Inventory, currency, and quest invariants tested.
- [ ] Crash-boundary and retry tests pass.
- [ ] Content bundle is approved and validated.

**Security**

- [ ] TLS, session revocation, CSRF, and WebSocket Origin checks verified.
- [ ] Staff permissions and production access reviewed.
- [ ] Dependency/secret scans reviewed with explicit exceptions.
- [ ] Abuse limits and slow-client controls exercised.

**Operations**

- [ ] Load test covers the approved admission limit.
- [ ] Restore drill meets the approved RPO/RTO or exceptions are accepted.
- [ ] Exclusive ownership and failover procedure tested.
- [ ] Dashboards, actionable alerts, and runbooks available.
- [ ] Drain, rollback, and forward-fix procedures rehearsed.
- [ ] Release owner records a go/no-go decision.

### 23.5 Living document ownership

| Document | Purpose | Owner |
|---|---|---|
| README | Build, run, configure, use | Engineering |
| DESIGN | Architecture, invariants, contracts, rationale | Engineering lead |
| SECURITY | Threat model, identity, secrets, reporting | Security owner |
| Gameplay rules | Authoritative player-facing and internal rules | Design lead |
| Content guide | IDs, schema, validation, publication | Content lead |
| API/protocol docs | Actual endpoints and message schemas | Engineering |
| Runbooks | Deployment, recovery, incidents | Service owner |
| Community policy | Conduct, enforcement, appeals | Community lead |

A behavior-changing pull request updates the relevant documentation in the same pass. Documentation is part of implementation, not a later milestone.

## 24. Common failure modes

| Failure | Why it hurts | Better approach |
|---|---|---|
| Build a universal engine before a playable game | Architecture grows without user evidence | Deliver a complete vertical slice |
| Treat every state change as periodic autosave | Valuable assets can be duplicated or lost | Classify durability and transact valuable mutations |
| Add microservices immediately | Distributed failures obscure basic game logic | Modular monolith with explicit boundaries |
| Let clients submit health/inventory values | Modified clients can cheat | Accept intent; compute outcomes server-side |
| Use unbounded queues | Slow clients or floods exhaust memory | Bounded mailboxes and explicit overload policies |
| Expose raw production database tools to staff | Mistakes bypass invariants and audit | Typed, scoped administrative commands |
| Consider CORS sufficient security | CSRF and WebSocket misuse remain | Authentication, authorization, CSRF, Origin checks |
| Turn on arbitrary scripting | Builders gain unintended host capabilities | Declarative actions or tightly bounded sandbox APIs |
| Assume a backup means recoverability | Corruption and missing keys appear during crisis | Isolated, measured restore drills |
| Scale replicas without world ownership | Split-brain simulation and duplicate assets | Ownership, fencing, and tested handoff |
| Expand world size instead of fixing the loop | More content amplifies weak mechanics | Playtest and repair the core experience |
| Launch without moderation staffing | Abuse overwhelms trust and retention | Fund the community operation from the beginning |

## 25. First thirty days

### Week 1: Decide and constrain

- Name product, engineering, content, service, security, and community owners.
- Write the charter and MVP exclusions.
- Compare adopt/extend/custom using a small executable spike.
- Establish licensing and content ownership rules.
- Open the risk register and initial ADRs.

### Week 2: Establish the foundation

- Create the actual workspace and document verified build commands.
- Add CI, migration tests, and content validation.
- Implement authenticated session boundaries and bounded message handling.
- Draft the browser client and one connected room sequence.
- Establish synthetic staging data and a basic deployment pipeline.

### Week 3: Make the loop playable

- Implement movement, look, inventory, speech, and help.
- Add one NPC, encounter, quest, and reward.
- Persist valuable state with idempotent operations.
- Add logs, metrics, and first moderation actions.
- Run a small internal playtest and record observed problems.

### Week 4: Prove integrity and operation

- Test retries, concurrent item access, and crash boundaries.
- Rehearse deployment, drain, and restoration in isolation.
- Validate the content workflow with someone other than its author.
- Review accessibility and abuse controls.
- Re-estimate the remaining work using actual delivery evidence.

These are goals for an experienced team, not a guarantee that all functionality can be completed in a month. Carry forward unfinished integrity work explicitly rather than declaring the foundation complete.

## 26. Reference material and glossary

### Primary documentation to consult during implementation

The following are starting references, not a claim that a particular release/API was verified while authoring this guide. Consult the documentation matching the pinned versions in the actual project.

- Rust Book: <https://doc.rust-lang.org/book/>
- Cargo documentation: <https://doc.rust-lang.org/cargo/>
- Tokio tutorial: <https://tokio.rs/tokio/tutorial>
- Axum crate documentation: <https://docs.rs/axum/>
- SQLx crate documentation: <https://docs.rs/sqlx/>
- Askama documentation: <https://askama.readthedocs.io/>
- PostgreSQL documentation: <https://www.postgresql.org/docs/>
- OWASP Cheat Sheet Series: <https://cheatsheetseries.owasp.org/>
- WebSocket protocol, RFC 6455: <https://www.rfc-editor.org/rfc/rfc6455>
- Telnet protocol, RFC 854: <https://www.rfc-editor.org/rfc/rfc854>
- Evennia documentation: <https://www.evennia.com/docs/>

### Glossary

| Term | Meaning |
|---|---|
| MUD | Multi-User Dungeon; persistent multiplayer text world |
| Actor | State owner that processes messages under an explicit ordering model |
| Authoritative server | Server that determines valid gameplay state and outcomes |
| Definition | Versioned template for an item, room, NPC, or quest |
| Instance | A particular runtime or persistent realization of a definition |
| Domain event | A meaningful game result, such as movement or reward completion |
| Idempotency | Repeating a request does not repeat its durable effect |
| Fencing | Rejecting operations from a stale state owner |
| Outbox | Durable record used to reliably deliver post-commit side effects |
| PITR | Point-in-time database recovery |
| RPO | Maximum targeted data-loss window during recovery |
| RTO | Maximum targeted time to restore service |
| SLO | Measurable service-level objective |
| Vertical slice | A complete user outcome crossing all necessary system layers |
| Realm | Independently operated world or population partition |
| GMCP | Generic Mud Communication Protocol, an optional traditional-client extension |

---

## Final principle

**Build the game, the authoring workflow, and the operating organization together.** Start with one enjoyable, persistent, recoverable loop. Expand only when evidence shows the current world is coherent, its assets are safe, and the team can support the people playing it.
