# Embedded Distributed Datastore Build Plan

## Goal

Build a Rust-based embedded datastore that runs inside a local host process on laptops and phones instead of as a standalone database server. Each device should be able to:

- read and write data locally
- keep working offline
- sync changes with other devices later
- recover safely after crashes or restarts
- stay portable across Android, Apple platforms, Windows, and macOS

This document is a practical roadmap for how to start, what to build first, and what to read as the system gets more advanced.

## Working Assumptions

- The current Rust workspace lives at the repository root with these crates:
  - `crates/core`
  - `crates/storage`
  - `crates/host`
  - `crates/test-harness`
- Version 1 should be an embedded local-first datastore, not a full distributed database cluster.
- Version 1 should prefer correctness, durability, and debuggability over clever distributed behavior.

## First Recommendation

Do not start by building:

- a SQL engine
- a general-purpose query planner
- full peer-to-peer mesh sync
- automatic conflict-free collaboration for every data type
- a cross-platform UI

Start by building a reliable local store with a change log. Then sync two nodes. Then harden. Then expose mobile and desktop bindings.

## Immediate Cleanup

Before adding real functionality, clean up the workspace itself.

### 1. Make the repository structure sane

Right now the member crates contain nested `.git` directories. For a normal Cargo workspace, you usually want one Git repository at the workspace root, not one repo per crate. Decide whether you want:

- one repo for the whole `Datastore` workspace, or
- genuinely separate repos for each crate

For this project, one workspace-level repo is the better default.

### 2. Add a docs area

Create a `docs/` folder later for:

- architecture decisions
- invariants
- protocol notes
- storage format notes
- failure model notes

### 3. Write down the invariants early

Before building features, write one short document that answers:

- What is a committed write?
- What does durable mean?
- Can two devices edit the same record concurrently in V1?
- How are conflicts detected?
- Is the sync model eventually consistent, single-writer, or leader-based?
- What should happen after a crash in the middle of a write?

If these stay fuzzy, the codebase will drift.

## Product Scope for Version 1

Choose a deliberately narrow V1:

- data model: key-value or document store
- storage model: append-only log plus materialized state
- sync model: two-node sync or device-to-hub sync
- conflict model: explicit conflict detection, not “magic merge”
- API model: library-first, with a small internal host process for testing

Recommended V1 shape:

- one local embedded node per device
- one logical collection or namespace system
- one simple replication protocol
- one explicit versioning model per record

## Recommended Architecture

Treat the system as four layers.

### Layer 1: Domain and invariants

This should live in `crates/core`.

Responsibilities:

- IDs and versions
- timestamps or logical clocks
- record metadata
- domain errors
- change representation
- conflict markers

This crate should avoid OS-specific behavior and networking.

### Layer 2: Local persistence

This should live in `crates/storage`.

Responsibilities:

- append-only commit log
- snapshots or compacted state
- crash recovery
- checksums and corruption detection
- indexing for reads

This is the most important crate in the early project.

### Layer 3: Local host runtime

This should live in `crates/host`.

Responsibilities:

- loading the store
- exposing an internal API or local RPC surface
- starting background tasks
- lifecycle and shutdown
- triggering compaction, snapshotting, and sync jobs

This should stay thin. It should orchestrate rather than contain storage logic.

### Layer 4: System testing

This should live in `crates/test-harness`.

Responsibilities:

- two-node and multi-node simulations
- restart tests
- partial failure tests
- delayed delivery and duplicate delivery tests
- corruption and recovery tests

For this kind of software, the test harness is not optional. It is part of the product.

## What To Add Later

After the first stable local engine, add:

- `crates/sync`
- `crates/transport`
- `crates/bindings`

Do not add them until the local persistence path is trustworthy.

## Development Phases

## Phase 0: Learn Just Enough Rust

### Objective

Get comfortable enough with Rust to build libraries, not just toy binaries.

### What to learn

- ownership and borrowing
- `Result` and error handling
- structs, enums, and traits
- modules, crates, and workspaces
- tests
- basic async concepts

### Exit criteria

You can explain:

- who owns a value
- when a borrow ends
- when to return `Result`
- when code belongs in a library crate vs a binary crate

### Read first

- The Rust Book: <https://doc.rust-lang.org/book/>
- Rust docs overview: <https://doc.rust-lang.org/>
- Cargo workspaces: <https://doc.rust-lang.org/cargo/reference/workspaces.html>
- Comprehensive Rust: <https://google.github.io/comprehensive-rust/>

### Read after basics

- Rust Reference: <https://doc.rust-lang.org/reference/>
- Rust for Rustaceans: <https://rust-for-rustaceans.com/>
- Rustonomicon: <https://doc.rust-lang.org/nomicon/>

## Phase 1: Define the Data Model and Consistency Story

### Objective

Pick a simple model you can actually implement and reason about.

### Recommended choice

Start with a document or key-value model instead of SQL.

Each record should have metadata such as:

- record ID
- version
- last modified logical time
- origin replica or device ID
- tombstone status if deleted

### Decisions to make

- Do writes overwrite values or append events?
- Is the system last-writer-wins in V1?
- Are concurrent edits rejected, flagged, or merged?
- Do you want per-record causality or a simpler version counter?

### Suggested V1 consistency model

Use explicit record versions and reject or flag concurrent conflicting writes. This is much safer for a first version than trying to build a general CRDT system immediately.

### Exit criteria

You have a one-page write-up explaining:

- your record shape
- your metadata shape
- your delete behavior
- your conflict policy
- your sync assumptions

### Read

- Designing Data-Intensive Applications: <https://dataintensive.net/>
- Jepsen consistency models guide: <https://jepsen.io/consistency/models>
- Convergence article by Martin Kleppmann: <https://martin.kleppmann.com/2022/11/01/convergence-cacm.html>

## Phase 2: Build the Local Storage Engine

### Objective

Make one device reliable before introducing distribution.

### Features to build

- append-only write log
- in-memory state reconstruction from log
- checkpoint or snapshot support
- basic indexes for lookup
- checksums for records or blocks
- deterministic recovery on restart

### Design guidance

Prefer an append-only log first because it simplifies:

- recovery
- auditing
- replication
- testing

You can compact or snapshot later to improve startup and storage cost.

### Failure cases to design for

- process dies after write begins
- process dies after log append but before index update
- partial write
- duplicate replay on restart
- corrupted entry

### Exit criteria

You can:

- write data
- restart the process
- rebuild state correctly
- detect corrupted or incomplete data
- prove with tests that replay is deterministic

### Read

- SQLite architecture: <https://www.sqlite.org/arch.html>
- SQLite atomic commit: <https://sqlite.org/atomiccommit.html>
- Database Internals by Alex Petrov: <https://www.databass.dev/>

## Phase 3: Define the Internal API Boundary

### Objective

Separate the storage core from the host process.

### What to design

- storage API for reads and writes
- transaction or batch boundary
- change feed API
- snapshot/load API
- admin or diagnostic API

### Guidance

Keep the API library-first. Your `host` crate should call into the core libraries rather than own business logic.

### Exit criteria

Another crate can use the datastore without reaching into storage internals.

### Read

- Cargo workspaces: <https://doc.rust-lang.org/cargo/reference/workspaces.html>
- Rust traits and modules in the Rust Book: <https://doc.rust-lang.org/book/>
- Rust Reference: <https://doc.rust-lang.org/reference/>

## Phase 4: Add the Local Host Runtime

### Objective

Run the embedded datastore inside an internal server process on a laptop first.

### Features to build

- process startup and shutdown
- configuration loading
- local request handling
- background workers
- graceful flush on shutdown
- structured logs and tracing

### Guidance

The host runtime should do these jobs:

- own process lifecycle
- schedule background work
- expose diagnostics

It should not become the place where storage correctness lives.

### Exit criteria

You can run one local node, restart it, observe it, and test it through a stable interface.

### Read

- Tokio tutorial: <https://tokio.rs/tokio/tutorial>
- Tokio async deep dive: <https://tokio.rs/tokio/tutorial/async>
- Tokio graceful shutdown: <https://tokio.rs/tokio/topics/shutdown>
- Tokio tracing topic: <https://tokio.rs/tokio/topics/tracing>
- `tracing` crate docs: <https://docs.rs/tracing>

## Phase 5: Add Change Tracking for Replication

### Objective

Represent local mutations in a form that can be replayed elsewhere.

### Features to build

- operation IDs
- replica IDs
- monotonic sequence numbers or logical clocks
- change serialization
- idempotent apply behavior

### Core rule

A replicated change should be safe to:

- receive late
- receive twice
- receive out of order if your protocol allows it

### Exit criteria

You can export a change stream from one node and replay it onto another clean node.

### Read

- Raft home page: <https://raft.github.io/>
- Raft paper: <https://raft.github.io/raft.pdf>
- Designing Data-Intensive Applications: <https://dataintensive.net/>

## Phase 6: Sync Two Nodes

### Objective

Get two devices to exchange changes correctly.

### Start with

- one laptop node and one second node
- manual peer configuration
- pull or push-pull sync
- one namespace or collection

### Do not start with

- peer discovery
- N-way sync
- automatic conflict-free merges for everything

### Questions to answer

- How does a node know what the other node has already seen?
- How are duplicate changes ignored?
- What happens if both devices edit the same record?
- Do you want leader-based replication or multi-writer eventual sync?

### Recommended V1

Use simple multi-writer replication with explicit conflict reporting, or a single-writer-per-record rule. Either is easier to reason about than “invisible merge semantics.”

### Exit criteria

You can:

- sync two nodes repeatedly
- stop and restart either node
- avoid duplicate application
- surface conflicts deterministically

### Read

- Local-first software article: <https://www.inkandswitch.com/essay/local-first/>
- CRDT resources list: <https://crdt.tech/resources>
- Automerge concepts: <https://automerge.org/docs/reference/concepts/>

## Phase 7: Decide Whether You Really Need CRDTs

### Objective

Avoid overengineering.

### Recommendation

Do not make CRDTs your default plan unless your product truly needs:

- concurrent multi-device editing of the same logical object
- automatic merges without user intervention
- high offline availability with collaborative updates

For many applications, explicit conflict detection is enough for V1.

### If you later need CRDTs

Study:

- Local-first software: <https://www.inkandswitch.com/essay/local-first/>
- Automerge docs: <https://automerge.org/>
- Strong eventual consistency proof work: <https://martin.kleppmann.com/2017/07/07/isabelle-crdt-proof.html>

## Phase 8: Add Cross-Platform Bindings

### Objective

Expose the Rust core to mobile and desktop hosts without rewriting the datastore.

### Strategy

Keep the core datastore in Rust and expose thin foreign-language bindings for:

- Swift on Apple platforms
- Kotlin on Android
- optional C ABI or C# integration for desktop apps

### Recommendation

Use a library boundary rather than embedding platform details in the core crates.

### Exit criteria

The same Rust core can be loaded by at least one foreign host language while preserving storage behavior.

### Read

- UniFFI guide: <https://mozilla.github.io/uniffi-rs/latest/>
- UniFFI foreign-language bindings: <https://mozilla.github.io/uniffi-rs/latest/tutorial/foreign_language_bindings.html>
- UniFFI prerequisites: <https://mozilla.github.io/uniffi-rs/latest/tutorial/Prerequisites.html>

## Phase 9: Hardening, Observability, and Abuse Testing

### Objective

Turn the prototype into a trustworthy system.

### Tests to add

- restart during write
- restart during compaction
- replay old changes
- duplicate message delivery
- out-of-order delivery
- dropped message recovery
- corrupted log segment
- disk-full behavior
- clock skew if wall-clock timestamps exist

### Metrics and diagnostics

- queue depth
- compaction time
- recovery time
- applied change count
- rejected change count
- conflict count
- last successful sync time

### Read

- Jepsen consistency models: <https://jepsen.io/consistency/models>
- `tracing` docs: <https://docs.rs/tracing>
- Tokio tracing topic: <https://tokio.rs/tokio/topics/tracing>

## Suggested Ownership of the Current Crates

## `crates/core`

Own:

- record schema primitives
- metadata
- clocks or versions
- errors
- portable traits

Do not put in here:

- file I/O
- sockets
- Tokio runtime details

## `crates/storage`

Own:

- on-disk log format
- snapshot format
- indexes
- recovery logic
- compaction

Do not put in here:

- API transport
- device discovery

## `crates/host`

Own:

- configuration
- runtime startup
- admin surface
- task orchestration

Do not put in here:

- low-level record format
- conflict resolution rules

## `crates/test-harness`

Own:

- simulation helpers
- deterministic distributed tests
- restart and corruption scenarios

This crate should eventually become one of the most valuable parts of the workspace.

## Recommended First Milestones

## Milestone 1: Single-Node Durable Store

Done means:

- local writes persist
- restart recovery works
- deterministic tests exist

## Milestone 2: Change Feed

Done means:

- every committed mutation produces a replayable change
- another node can consume the change log format

## Milestone 3: Two-Node Sync

Done means:

- nodes exchange changes
- conflicts are detected deterministically
- repeated sync is idempotent

## Milestone 4: One Mobile or Foreign Binding

Done means:

- Rust core is called from one host outside Rust
- behavior matches laptop host tests

## A 12-Week Roadmap

## Weeks 1-2

- clean up repository layout
- write architecture notes
- learn Rust workspace basics
- define data model and invariants

## Weeks 3-4

- implement append-only local persistence
- add recovery tests
- add checksums or corruption checks

## Weeks 5-6

- define storage API
- build minimal host runtime
- add tracing and diagnostics

## Weeks 7-8

- add change tracking
- define replication message format
- simulate export/import between nodes

## Weeks 9-10

- implement two-node sync
- test duplicates, restarts, and conflict scenarios

## Weeks 11-12

- refine compaction and recovery
- document protocol invariants
- evaluate first mobile or foreign binding path

## Reading Order

If you want the shortest reading path that still supports this project, use this order:

1. The Rust Book
2. Cargo workspaces
3. Tokio tutorial
4. SQLite architecture
5. SQLite atomic commit
6. Designing Data-Intensive Applications
7. Raft paper
8. Local-first software
9. Jepsen consistency models
10. UniFFI guide

If you want a deeper Rust path after the basics:

1. Comprehensive Rust
2. Rust for Rustaceans
3. Rust Atomics and Locks: <https://mara.nl/atomics/>
4. Rustonomicon

## What Success Looks Like

After the first serious iteration, you should have:

- a reliable embedded local store
- deterministic restart behavior
- a replayable change log
- two-node sync with clear conflict semantics
- a codebase that is still small enough to reason about

That is the right point to decide whether to invest in:

- richer queries
- stronger consistency
- CRDT-based collaboration
- peer discovery
- encryption at rest
- mobile-first bindings

## Final Advice

The highest-risk parts of this project are not syntax or language features. They are:

- durability semantics
- recovery correctness
- replication correctness
- conflict semantics
- testability under failure

Build those before chasing features.

If you keep the storage core small, explicit, and well-tested, the rest of the system can grow around it without collapsing under its own complexity.
