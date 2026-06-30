# Reusable DesCartes Bug-Finding Prompt for Rust Projects

You are testing this Rust project with DesCartes.

## Primary Goal

Find real concurrency, scheduling, cancellation, timeout, resource-accounting, or lifecycle violations.

Optimize for **bug yield**, not broad test coverage.

Do not merely add tests. Do not build abstract models unless explicitly justified. The default method is:

> copy real source → mechanically swap runtime/imports → preserve semantics → write source-backed invariants → run deterministic schedule exploration → classify failures against actual contracts.

## First Phase: Understand the Target and DesCartes Fit

Before writing tests, read:

1. The project source.
2. The project docs, comments, examples, and tests.
3. Public API contracts.
4. DesCartes docs needed for this project:

   * `des-core` / `descartes_core`
   * `des-tokio` / `descartes_tokio`
   * `des-tower`
   * `des-tonic`
   * `des-components`
   * scheduler policies
   * simulated time
   * seeded replay/exploration if available.

Do not assume every Rust async project is Tokio-only.

Choose the DesCartes facade based on the actual abstraction:

* Use `des-tokio` for Tokio-style async tasks, channels, timers, mutexes, semaphores, and `select!`.
* Use `des-tower` only when testing real Tower service stacks or copied Tower-style service state machines.
* Use `des-tonic` only when testing tonic-like RPC or simulated transport paths.
* Use `des-core` / `des-components` for lower-level simulation or environment modeling, not as a replacement for the target implementation.

## Hard Rules

1. Do not write a behavioral model of the target library and test the model.
2. Do not replace the target state machine with a DesCartes component.
3. Do not classify behavior as a bug unless it violates source-backed or documented behavior.
4. Do not use strict fairness, exact ordering, or timing expectations as bug oracles unless explicitly guaranteed.
5. Fakes are allowed only at external boundaries:

   * network
   * database
   * filesystem
   * process
   * clock
   * external peer
   * external service
6. Local compatibility shims are allowed only when DesCartes lacks a small API needed by copied source. Failures near shims are TEST BUG until audited.

## Phase 1: Inventory the Concurrency Surface

Create a candidate target table.

For each source file/module, identify:

* async tasks
* spawned workers
* event loops
* timers, sleeps, intervals
* timeouts
* retries and backoff
* bounded queues
* channels
* streams
* `select!` branches
* dropped losing branches
* request/response routing
* cloned handles
* shared state
* secondary indexes
* semaphores
* permits
* pools
* leases
* resources
* background cleanup
* purge loops
* refresh loops
* shutdown/drain/close paths
* cancellation paths
* retry/reconnect paths
* TTL/expiry/cache eviction
* pub/sub or broadcast fanout
* slow consumer handling
* Tower `poll_ready` / `call` state machines
* RPC stream open/close paths
* multiplexed clients with in-flight work.

## Phase 2: Rank Targets by Bug Yield

Score each candidate target.

High-value signals:

* +5 cancellation can leak capacity, permit, resource, waiter, request id, or connection slot
* +5 timeout races with success, failure, cleanup, or retry
* +5 resource accounting can diverge from reality
* +5 request/response routing can misdeliver, drop, duplicate, or strand responses
* +4 bounded queue, backpressure, or slow-consumer behavior exists
* +4 background worker performs cleanup, retry, purge, refresh, or drain
* +4 shutdown/close races with in-flight work
* +4 retry/reconnect while work is in-flight
* +4 `select!` drops a losing branch with side effects
* +4 cloned handles share mutable state
* +4 waiter queue or notification logic exists
* +3 simulated time can expose stale timers, expiry, retry, or backoff errors
* +3 prior issues mention hang, deadlock, timeout, leak, race, cancellation, lost message, stale state, or shutdown
* +3 small enough to copy and mechanically adapt
* +3 strong public contract exists
* +2 existing tests already encode expected behavior

Low-value signals:

* -5 requires real sockets/processes/files to test meaningfully
* -5 invariant depends on undocumented fairness
* -5 invariant depends on exact ordering not promised
* -5 requires rewriting target semantics
* -4 source is too large to isolate
* -4 failure would mainly test fake environment behavior
* -3 mostly synchronous code
* -3 thin wrapper with little state
* -3 already heavily tested primitive with narrow contract
* -2 no clear public contract

Prioritize targets with the highest score.

Prefer bug-shaped state machines over merely interesting concurrent code.

## Phase 3: Extract Source-Backed Invariants

Before writing scenarios, create an invariant table.

For each invariant include:

* target name
* source path
* function/type/state machine
* relevant source comment, doc, test, or API contract
* invariant statement
* resource or state being protected
* cancellation point
* timeout/simulated-time dependency
* race/interleaving that may violate it
* expected failure symptom
* classification:

  * STRONG
  * WEAK
  * INVALID BUG ORACLE

### STRONG Invariant

Use STRONG when the property is supported by:

* public docs
* source comments
* upstream tests
* API semantics
* memory/resource safety expectations
* request/response correctness
* permit/resource lifecycle correctness
* no-loss/no-duplication guarantee
* shutdown/drain guarantee.

STRONG failures may become BUG candidates after audit.

### WEAK Invariant

Use WEAK when the property is useful but not clearly guaranteed:

* fairness
* approximate ordering
* observability
* performance expectation
* eventual cleanup without explicit guarantee
* behavior inferred but not documented.

WEAK failures require contract review.

### INVALID BUG ORACLE

Use INVALID BUG ORACLE when the property assumes:

* stricter ordering than promised
* fairness not promised
* timing not promised
* caller misuse
* behavior stronger than docs
* fake-I/O artifact
* shim artifact
* abstract-model artifact.

Do not classify these as bugs.

## Phase 4: Choose the Copy Boundary

Create or use a `descartes-tests/` crate.

Copy relevant source files into `descartes-tests/src/`.

Preserve implementation semantics.

Allowed changes:

* visibility changes only as needed for tests
* import swaps from Tokio/runtime APIs to DesCartes-compatible APIs
* external I/O replaced with deterministic fakes
* local shims for missing tiny APIs
* test-only tracing hooks if they do not change behavior
* comments documenting every deviation.

Not allowed:

* replacing the target state machine with a model
* simplifying behavior to make testing easier
* changing cancellation/drop semantics
* changing queue/resource accounting logic
* changing timeout/retry logic
* changing ordering semantics
* replacing target internals with DesCartes components.

## Phase 5: Design Deterministic Fakes

Fakes should model only the external environment, not the target implementation.

Common fake knobs:

* immediate success
* delayed success
* permanent failure
* fail N times then succeed
* pending forever
* cancellation at specific poll point
* slow consumer
* dropped receiver
* closed sender
* timeout boundary
* partial response
* duplicate response
* lost external response
* reconnect success/failure
* validation success/failure
* resource marked broken/expired
* simulated time advancement.

Each fake should emit trace events.

Useful trace events:

* task_started
* operation_started
* operation_pending
* operation_cancelled
* operation_timeout
* operation_success
* operation_error
* resource_created
* resource_checked_out
* resource_returned
* resource_discarded
* resource_reused
* waiter_registered
* waiter_cancelled
* waiter_woken
* request_sent
* response_received
* response_delivered
* response_dropped
* background_worker_started
* background_worker_stopped
* shutdown_started
* shutdown_completed.

## Phase 6: Build Scenarios from High-Yield Invariants

Start with small, explainable schedules.

Avoid beginning with random fuzz-style actions.

For each scenario document:

* test name
* source-backed invariant
* invariant strength
* target source path
* copied files
* fake environment behavior
* cancellation point
* timeout point
* scheduler-sensitive race
* assertion
* expected trace/state on failure.

High-yield scenario templates:

### Cancellation while waiting

A task begins an operation and becomes a waiter. Drop/cancel it before the resource/event becomes available.

Check:

* no leaked waiter
* no leaked capacity
* no future starvation
* no stale delivery to cancelled task.

### Timeout racing with success

An operation times out at nearly the same simulated time as success/failure.

Check:

* no double completion
* no lost resource
* no stale state
* no leaked in-flight slot.

### Dropped losing `select!` branch

A `select!` branch performs partial work, then loses and is dropped.

Check:

* no resource leak
* no lost notification
* no partially registered waiter
* no stranded request.

### Return/checkin while waiter cancels

A resource is returned while a waiting task is cancelled.

Check:

* returned resource is not lost
* cancelled waiter does not receive it
* a live waiter or future caller can make progress.

### Background cleanup racing with foreground operation

A cleanup/retry/purge/refresh worker races with checkout, get, send, call, close, or update.

Check:

* no stale state
* no double delete
* no missed cleanup required by contract
* no use-after-expiry if contract forbids it.

### Shutdown while work is in flight

Shutdown/close/drain races with pending requests, queued messages, retries, or streams.

Check:

* pending work resolves according to contract
* no silent loss if delivery is guaranteed
* no hang
* no post-shutdown acceptance if forbidden.

### Retry/reconnect while request is in flight

A retry/reconnect path runs while earlier work is pending.

Check:

* no duplicate response delivery
* no response routed to wrong request
* no old connection state reused incorrectly
* no permanent poisoned state after transient failure.

### Bounded queue under slow consumer

Producer fills queue while consumer is slow, cancelled, or closed.

Check:

* backpressure behavior matches contract
* no accepted message is silently lost unless documented
* close wakes blocked producers/consumers.

## Phase 7: Run Schedule Exploration

For each scenario:

1. Run default/FIFO scheduler first.
2. Run seeded randomized policies.
3. Use available policies such as:

   * UniformRandomFrontierPolicy
   * UniformRandomReadyTaskPolicy
4. Run at least 500 seeds per scenario.
5. Run 2,000+ seeds for high-yield races.
6. Vary parameters, not only seeds.

Parameter dimensions:

* resource limit / pool size / queue bound
* number of callers
* number of waiters
* number of cloned handles
* timeout duration
* retry count
* backoff duration
* connect/send/receive delay
* cancellation point
* drop point
* close/shutdown timing
* background worker timing
* validation failure
* slow consumer speed
* simulated time advancement.

If available, use replay/minimization tooling to reduce traces.

## Phase 8: Assertions and Test Oracle

Maintain a test-side oracle for observable behavior.

Track:

* live tasks
* cancelled tasks
* pending operations
* completed operations
* timed-out operations
* resources created
* resources checked out
* resources returned
* resources discarded
* in-flight requests
* delivered responses
* dropped responses
* waiter count
* queue length if observable
* shutdown state.

Common assertions:

* no resource is checked out by two tasks simultaneously
* cancelled task does not receive successful completion
* timeout does not later also complete successfully
* accepted request is not lost unless contract permits loss
* response is delivered to the correct request
* resource capacity is eventually restored after cancellation/timeout
* broken/expired resource is not reused if contract forbids it
* close/shutdown wakes or resolves pending waiters according to contract
* background worker does not over-create or over-delete resources
* no permanent starvation when the contract implies progress and the fake environment is healthy.

Avoid assertions based only on:

* exact task order
* strict fairness
* exact number of polls
* exact timing unless simulated time contract requires it
* behavior stronger than public docs.

## Phase 9: Failure Audit

Do not immediately call a failure a bug.

For each failure, report:

* scenario name
* seed
* scheduler policy
* parameter configuration
* exact assertion
* trace
* copied source paths
* copy-diff notes
* fake configuration
* shim configuration
* source path causing suspected behavior
* invariant strength
* contract evidence
* upstream test evidence if relevant.

Verdict:

* BUG:
  A documented/source-backed/API-required guarantee is violated by copied source
  under a realistic fake environment.

* KNOWN LIMITATION:
  Behavior is allowed, documented, or intentionally weaker than the test oracle.

* TEST BUG:
  Harness, fake, shim, copied source, or assertion changed/assumed semantics.

* NEEDS CONTRACT REVIEW:
  Failure is plausible but contract support is insufficient.

## Phase 10: If No Bugs Are Found

Produce an invariant coverage report before continuing.

Include:

* candidate targets considered
* target scores
* invariants tested
* invariants not tested
* STRONG/WEAK/INVALID classification
* scenarios run
* seeds run
* parameters varied
* upstream tests compared
* DesCartes limitations encountered
* whether target is now low-yield
* recommendation:

  * continue same target
  * move to another module
  * move to another project.

## Project Selection Heuristic

When choosing the next Rust project, prioritize libraries with:

1. async resource pools
2. connection managers
3. retry/backoff/rate-limit logic
4. cancellation-sensitive futures
5. bounded queues
6. background maintenance tasks
7. TTL/cache cleanup
8. request/response multiplexing
9. reconnect logic
10. shutdown/drain semantics.

High-yield project categories:

* connection pools
* async client libraries with pooling/reconnect
* cache/TTL libraries
* retry/backoff/rate-limit libraries
* service buffers/load-shedding middleware
* pub/sub or broadcast systems
* RPC clients/servers
* bounded async queue abstractions
* distributed-system components with fakeable I/O.

Lower-yield categories:

* pure data structures
* thin wrappers
* fully synchronous libraries
* heavily tested core primitives
* projects requiring real sockets/processes for meaningful behavior
* huge systems where the core state machine cannot be isolated.

## Deliverables

1. Candidate concurrency target inventory with source paths.
2. Bug-yield score table.
3. Invariant table with STRONG/WEAK/INVALID classification.
4. Selected target and rationale.
5. Copied-source DesCartes harness.
6. Mechanical import/runtime swap notes.
7. Deterministic fakes at external boundaries.
8. Scenario list.
9. FIFO run results.
10. 500+ seed randomized run results per scenario.
11. 2,000+ seed results for highest-yield races.
12. Failure reports with BUG / KNOWN LIMITATION / TEST BUG / NEEDS CONTRACT REVIEW.
13. If no bugs are found, invariant coverage report and next-target recommendation.
