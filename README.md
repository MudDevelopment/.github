# MUD Development

**Building persistent worlds through collaborative game design, engineering, and community stewardship.**

## Overview

MUD Development is a collaborative development initiative focused on creating and operating a **Multi-User Dungeon (MUD)**: a persistent, multiplayer, primarily text-based game where players explore a shared world, interact with characters and objects, complete quests, and shape their experience through meaningful choices.

The effort brings together software developers, game designers, world builders, writers, testers, operators, and community contributors. Its purpose is not simply to create a game server, but to establish the tools, practices, and organizational responsibilities needed to build an engaging world and sustain it over time.

The project combines the imagination and social interaction of classic MUDs with modern engineering practices, accessible browser-based interaction, secure account management, and reliable live-service operations.

> **Project status:** This README describes the proposed mission, scope, and development direction. The organizational guide establishes a planning baseline; implementation progress, repository structure, and deployment readiness have not been verified. Planned capabilities below should not be read as already available features.

## Our Mission

Create a compelling, accessible, and maintainable multiplayer text world that rewards exploration, encourages community, and provides a dependable foundation for continued development.

We aim to:

- **Deliver a complete player experience:** Connect exploration, encounters, progression, and social interaction into a coherent game.
- **Empower creators:** Give designers, writers, and builders a safe, repeatable way to develop and publish world content.
- **Build for reliability:** Protect persistent player progress and make the service observable, recoverable, and supportable.
- **Encourage collaboration:** Use clear ownership, peer review, shared documentation, and constructive feedback.
- **Support a healthy community:** Treat accessibility, fair play, moderation, and player safety as core product responsibilities.

## What We Are Building

The development effort encompasses three closely related areas:

| Area | Purpose |
|---|---|
| **The game** | A shared text-based world with exploration, characters, items, encounters, quests, progression, and communication. |
| **The creation tools** | Workflows for authoring, validating, reviewing, testing, and publishing world content. |
| **The live service** | Account management, administration, moderation, deployment, monitoring, backup, recovery, and ongoing support. |

The initial goal is a **small, complete, playable world**, rather than a large collection of unfinished systems. Setting, audience, monetization, and final gameplay rules require explicit product decisions before they become release commitments.

### Initial Gameplay Scope

The proposed first release focuses on:

- Account registration, login, recovery, and character creation.
- Connected rooms, descriptive locations, exits, objects, and non-player characters.
- Core commands for looking, moving, communicating, managing inventory, and interacting with the world.
- A limited combat and reward system.
- A tutorial and at least one complete quest.
- Persistent character progression and item ownership.
- Safe disconnect, reconnect, and server-restart behavior.
- Basic moderation and operational tooling.

Potential later additions include crafting, guilds, trading, player housing, richer NPC behavior, expanded builder tools, and support for traditional MUD clients. These are roadmap candidates, not commitments for the initial release.

## How the Organization Works

MUD Development is organized around shared outcomes and clear responsibilities. Team members may cover multiple roles, but each area needs an accountable owner.

| Responsibility | Focus |
|---|---|
| **Product and project leadership** | Vision, priorities, scope, funding, milestones, and release decisions. |
| **Game design** | Rules, progression, encounters, rewards, balance, and the player experience. |
| **Engineering** | Architecture, server and client development, persistence, integration, and developer tooling. |
| **World building and writing** | Locations, characters, narrative, quests, descriptions, and content consistency. |
| **Quality assurance** | Acceptance criteria, automated testing, playtesting, regression coverage, and release evidence. |
| **Security and operations** | Identity, access control, deployment, monitoring, incident response, and recovery. |
| **Community and moderation** | Conduct standards, player feedback, reports, appeals, and community health. |

Development should proceed through small, reviewable changes and regular playable demonstrations. Product, design, engineering, and operations decisions should be documented so contributors understand both what is being built and why.

## Technical Direction

The reference approach is a **Rust-based modular monolith**: one deployable application with clear internal boundaries, rather than a collection of independently deployed gameplay services.

| Component | Proposed direction |
|---|---|
| Core implementation | Rust |
| Asynchronous runtime | Tokio |
| Web and application interfaces | Axum |
| Browser gameplay connection | Secure WebSockets |
| Durable storage | PostgreSQL with SQLx |
| Server-rendered web pages | Askama and HTML |
| World content | Versioned, validated content files |
| Initial hosting model | Linux, TLS-enabled ingress, and one authoritative world process |

These choices are a reference architecture, not a verified dependency inventory. The team should confirm the build-versus-adopt decision before committing to a custom engine.

### Architectural Principles

- **Server authority:** The server validates commands and controls gameplay outcomes; clients do not dictate inventory, rewards, or combat results.
- **Explicit state ownership:** World-state changes pass through a defined simulation owner to reduce concurrency errors.
- **Durable player value:** Important changes to items, currency, and progression receive appropriate transactional and recovery guarantees.
- **Separation of concerns:** Networking, game rules, persistence, content, and administration remain distinct and testable.
- **Bounded work:** Input sizes, command rates, queues, and background tasks have explicit limits.
- **Incremental growth:** Add distribution, scripting, and advanced systems only when validated requirements justify their complexity.

## Development Standards

The project should use the following standards from its first implementation milestone:

1. **Define acceptance criteria before implementation.** Specify intended behavior, failure cases, and what constitutes completion.
2. **Deliver vertical slices.** Build features across player interaction, game rules, persistence, and verification rather than leaving disconnected components unfinished.
3. **Review code and content.** Apply peer review to implementation, gameplay data, migrations, and operational changes.
4. **Test important guarantees.** Verify ownership, permissions, reward handling, reconnection, and recovery—not only successful command execution.
5. **Keep documentation current.** Changes to behavior, configuration, interfaces, or deployment update the relevant documentation in the same change.
6. **Report verification honestly.** Distinguish implemented, tested, proposed, and deferred work.
7. **Protect player trust.** Avoid exposing credentials or private messages in logs; use least-privilege access and auditable administrative actions.

Application binaries should live under `apps/`, with package, directory, and binary names aligned. Shared libraries should live under `crates/`. These are intended repository conventions, not a claim that those directories already exist.

## Roadmap

| Phase | Primary outcome | Completion evidence |
|---|---|---|
| **Foundation** | Agree on the game charter, ownership, engine approach, architecture, and working practices. | Approved scope and documented decisions. |
| **Playable vertical slice** | Connect, enter the world, explore, communicate, complete an encounter, and retain progress. | Demonstrable end-to-end play and persistence tests. |
| **Internal alpha** | Expand content and harden identity, moderation, administration, and developer tools. | Repeatable playtests and documented defect resolution. |
| **Closed beta** | Operate with invited players and validate deployment, recovery, capacity, and support procedures. | Operational readiness evidence and player feedback. |
| **Public readiness** | Decide whether the game and organization can support a broader audience. | Security review, tested recovery, sustainable support coverage, and an explicit release decision. |

Dates and staffing commitments should be established after technical discovery and early playtesting. Public launch is a readiness decision, not merely the completion of a feature list.

## Contributing

Contributions can include code, world content, writing, testing, accessibility feedback, documentation, and operational improvements.

A recommended contribution workflow is:

1. Read this overview and the organizational guide.
2. Agree on an issue or work item with a clear owner and acceptance criteria.
3. Confirm the current source, interfaces, and documented behavior before changing them.
4. Implement a focused change with appropriate tests or content validation.
5. Update affected documentation and describe any compatibility or migration impact.
6. Submit the change for review with a summary of what changed and how it was verified.

Repository-specific contribution instructions, review policies, conduct standards, and private security-reporting channels should be published as the project is established. Do not submit credentials, production player data, or assets without appropriate usage rights.

## Documentation

The [MUD Development Organizational Guide](MUD-Development-Organizational-Guide.md) provides the detailed blueprint for architecture, coding, persistence, content creation, security, testing, deployment, and live operations. Keep it alongside this README for the relative link to resolve.

As implementation proceeds, establish and maintain:

| Document | Purpose |
|---|---|
| `README.md` | Project overview and, once verified, contributor setup and usage instructions. |
| `DESIGN.md` | Implemented architecture, data flow, contracts, and key decisions. |
| `SECURITY.md` | Security policies, access boundaries, and private vulnerability reporting. |
| `CONTRIBUTING.md` | Local development, testing, review, and contribution requirements. |
| `CODE_OF_CONDUCT.md` | Contributor expectations and community conduct procedures. |
| Operational runbooks | Deployment, monitoring, backup, restore, and incident response. |

Except for this README and the companion guide, this table describes recommended documentation—not files confirmed to exist.

## Getting Started

For the organizational effort, begin by assigning accountable owners, agreeing on a one-page game charter, and defining the first playable vertical slice. Use the companion guide to turn those decisions into a prioritized backlog.

For implementation, first verify the repository and selected toolchain. Add exact build, test, configuration, and local-run instructions here only after they have been exercised against the actual project. No executable setup procedure is claimed by this planning-stage README.

## Licensing and Ownership

The project must explicitly establish licensing and contribution rights for source code, world content, documentation, and visual assets before external distribution. This README does not assign a software or content license. Any adopted engine, library, or third-party asset requires a compatibility and attribution review.

---

**Our guiding objective: build a world worth exploring, a codebase worth maintaining, and a community worth supporting.**
