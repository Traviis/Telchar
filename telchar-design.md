# Telchar Design Brief

**Status:** Draft

Telchar is a self-hosted Nix build gateway. It accepts ordinary Nix remote-build requests over SSH, applies admission control and scheduling, and dispatches each derivation to a compatible execution backend.

The project is named after Telchar, the renowned smith of Nogrod in Tolkien's legendarium. The name is thematic; public APIs and operator terminology should remain straightforward and domain-focused.

## Project summary

Telchar presents one stable Nix remote-builder endpoint to users:

```text
Stock Nix client
      │
      │ SSH + Nix worker protocol
      ▼
Telchar
      │
      ├── admission and quotas
      ├── queueing and scheduling
      ├── input and output transfer
      ├── execution tracking
      └── result and log streaming
             │
             ▼
      Execution backend
       ├── local Nix
       ├── static SSH builder
       ├── Nomad batch job
       └── future provider
```

Users keep using standard Nix commands and standard remote-builder configuration. They do not install a custom client, patch Nix, or learn the topology of the build fleet.

Telchar decomposes the system into small responsibilities:

- Nix remains responsible for evaluation and deciding which derivations are ready to build.
- Telchar accepts buildable derivations, enforces policy, and chooses an execution backend.
- A backend executes one derivation and reports its result.
- External infrastructure manages machine provisioning and autoscaling.
- An optional binary cache accelerates transfers and shares completed outputs, but is not required for correctness.

## Goals

### Transparent Nix integration

Telchar must work with unmodified, generally available Nix clients through the standard remote-builder interface.

A client should be able to use Telchar through normal configuration such as:

```conf
builders = ssh-ng://nix-builder@telchar.example.org x86_64-linux,aarch64-linux
```

The exact URI and supported systems are deployment-specific. Telchar must not require an experimental client-side protocol, patched Nix package, wrapper command, or custom evaluator.

### Central scheduling

Telchar should give all connected clients one shared scheduling point. The scheduler can therefore make decisions using global queue and execution state rather than the partial view available to each individual Nix daemon.

### Pluggable execution

Execution must be separate from scheduling. Telchar should be able to dispatch a derivation to:

- A local Nix installation.
- A persistent machine reachable over SSH.
- A one-shot Nomad batch job.
- A future backend without changing the client protocol or scheduler model.

Backends execute work. They do not own admission policy, user quotas, or global scheduling.

### External autoscaling

Telchar should expose enough demand and execution state for an external autoscaler, but it should not provision machines itself.

For example, a Nomad backend may create one pending batch job per derivation. Nomad and an external autoscaler can react to pending allocations by adding capacity. Telchar does not need to know whether that capacity comes from an AWS Auto Scaling Group, physical machines, another cloud, or a static cluster.

### Optional caching

A shared binary cache such as Attic should improve cache hits, staging, and output distribution when configured. Telchar must still function when no shared cache exists or when an optional cache is unavailable.

### Useful for individuals and organizations

The initial implementation should remain small enough for a single operator while establishing boundaries suitable for multiple engineers:

- Stable requester identity.
- Per-identity admission limits.
- Fair scheduling.
- Auditable executions.
- Backend isolation.
- No shared private client key.

## Non-goals

Telchar is not intended to:

- Evaluate flakes or Nix expressions.
- Replace `nix-daemon` on client machines.
- Act as a CI orchestrator.
- Receive source-control webhooks.
- Manage jobsets or build matrices.
- Provision or destroy compute instances.
- Implement an AWS autoscaler.
- Replace Nomad, Kubernetes, or another infrastructure scheduler.
- Require a particular binary cache.
- Replace all binary-cache software.
- Implement a new content-addressed object store in its first version.
- Schedule arbitrary shell commands.
- Expose interactive shell access to builders.
- Promise transparent migration of an in-progress build between executors.

## Why Telchar exists

Nix remote building provides a capable transport and execution protocol, but it is not a complete shared build-farm control plane.

Nix provides:

- Local evaluation.
- Derivation dependency traversal.
- Store-path transfer.
- Remote build delegation.
- Build logs and results.
- System and feature declarations.

A shared elastic build service additionally needs:

- Stable ingress.
- Global admission control.
- Requester quotas.
- Queue fairness.
- Backend selection.
- Central visibility into active work.
- Dynamic execution targets.
- Safe cancellation and draining.
- Infrastructure-neutral execution.
- Operational metrics and audit records.

Telchar fills this control-plane gap while retaining standard Nix clients.

## The Nix worker protocol

When Nix uses an `ssh-ng://` remote builder, it does not normally send a textual `nix build` shell command over SSH.

SSH starts a remote process, conventionally:

```text
nix-daemon --stdio
```

The local Nix daemon and remote process then exchange Nix's binary worker protocol over standard input and output. The protocol includes operations for:

- Version negotiation.
- Build options.
- Store-path validity queries.
- Path metadata queries.
- NAR uploads and downloads.
- Derivation build requests.
- Structured log and error frames.
- Build results and output metadata.

Telchar's SSH endpoint replaces `nix-daemon --stdio` with its own worker-protocol server:

```text
ForceCommand telchar serve-stdio
```

From the client's perspective, Telchar behaves as a remote Nix store and builder. Internally, Telchar can queue and dispatch the requested build elsewhere.

Worker-protocol compatibility is one of the highest-risk parts of the project. Telchar will selectively import the BSD-3-Clause-licensed `rio-nix` code from the archived rio-build repository where it fits, preserving its copyright, license, and source attribution. The imported code will be maintained inside the Telchar repository rather than consumed as a live dependency. Telchar should extend or replace individual pieces only when its requirements or compatibility tests justify doing so; it should not casually reimplement protocol framing, NAR streaming, version differences, and structured error handling from scratch.

## Core architectural principle

A submitted derivation is the unit of scheduling and execution.

In standard remote-builder mode, the local Nix daemon traverses the dependency graph and submits derivations when their dependencies are available. Telchar can therefore begin without implementing a full DAG evaluator or planner.

The initial model is:

```text
one BuildDerivation request
        │
        ▼
one Telchar queued build
        │
        ▼
one selected backend execution
        │
        ▼
one structured build result
```

A backend may use a persistent worker or create a one-shot allocation, but that difference remains behind the backend interface.

## Components

### SSH ingress

The SSH layer provides:

- Encryption.
- Server authentication.
- Client public-key or certificate authentication.
- Source address information.
- A restricted forced command.
- No shell, PTY, forwarding, or unrelated command execution.

Telchar should support ordinary SSH host keys. SSH host certificates may be configured, but must not be required.

Likewise, clients may authenticate with:

- Authorized public keys.
- OpenSSH user certificates.
- Certificates issued by Vault or another SSH CA.

The protocol and scheduler should depend on normalized requester identity, not a particular certificate issuer.

### Protocol gateway

The protocol gateway:

- Negotiates the Nix worker protocol.
- Receives store paths and derivation requests.
- Maintains the protocol session.
- Converts protocol operations into internal store and build operations.
- Streams logs and terminal results back to the client.
- Handles client disconnects without corrupting build state.

The gateway must not contain backend-specific Nomad or SSH scheduling logic.

### Gateway store

The first implementation should use a real Nix store owned or controlled by Telchar.

The gateway store is the correctness boundary for an active request. It holds:

- Uploaded derivations.
- Client-provided source and input paths.
- Inputs required by executors.
- Outputs collected from successful executions.
- Nix path metadata required to answer the client protocol.

A local Nix store avoids prematurely designing a custom CAS, NAR database, garbage collector, or path-info implementation.

A future implementation may introduce alternate storage, but the initial system should optimize for correctness and interoperability.

### Admission controller

Admission occurs before expensive execution is started.

It should enforce bounded limits such as:

- Maximum global queued requests.
- Maximum global running requests.
- Maximum queued requests per identity.
- Maximum running requests per identity.
- Optional request size or input-transfer limits.
- Optional architecture or feature policy.

A rejected request must receive a clear Nix-compatible error. It must not be silently dropped.

### Scheduler

The scheduler owns:

- Queue ordering.
- Fairness between identities.
- Backend compatibility filtering.
- Backend ranking.
- Dispatch.
- Execution state.
- Retry classification.
- Cancellation eligibility.

The initial scheduler should be deliberately modest. A reasonable first policy is round-robin across requester identities with per-identity concurrency limits, then backend selection by:

1. Required system.
2. Required features.
3. Administrative priority.
4. Available capacity.
5. Configured backend preference or cost.
6. Current active execution count.

This avoids both first-in global monopolization and premature work on a sophisticated weighted scheduler.

### Execution registry

Telchar needs durable or reconstructable state for:

- Request ID.
- Requester identity.
- Derivation path.
- Requested system and features.
- Queue time.
- Selected backend.
- Backend execution ID.
- Current state.
- Start and completion timestamps.
- Terminal classification.
- Output paths.
- Audit metadata.

A first implementation may use SQLite if one active Telchar instance is an explicit constraint. PostgreSQL is appropriate when high availability or multiple gateway replicas become requirements. Storage choice should follow actual deployment requirements rather than being abstracted prematurely.

### Backend interface

A backend should implement a narrow lifecycle:

```text
submit(request) -> execution
status(execution) -> pending | running | completed | failed
logs(execution) -> stream or cursor
cancel(execution)
collect(execution) -> build outputs and result metadata
```

The precise Rust API may evolve, but backend implementations should receive a normalized build request rather than protocol or SSH session objects.

A conceptual request contains:

```text
BuildRequest
├── request ID
├── derivation path
├── target Nix system
├── required system features
├── build options
├── input closure references
├── requester identity
└── cancellation state
```

A conceptual result contains:

```text
BuildOutcome
├── terminal status
├── output paths and metadata
├── build log reference
├── backend and executor identity
├── timing information
└── infrastructure/build failure classification
```

Backend names and APIs should describe domain behavior. Avoid implementation-history names such as `NewBackend`, `LegacyExecutor`, or `UnifiedRunner`.

## Backends

### Local backend

The local backend executes the derivation using a local Nix daemon or store.

Primary uses:

- Development.
- Protocol integration tests.
- Small installations.
- A simple reference implementation for the backend contract.

It should invoke Nix with structured arguments or a library API. It must not interpolate untrusted derivation paths into a shell command.

### Static SSH backend

The SSH backend delegates to a persistent Nix machine.

The machine may expose the ordinary Nix worker protocol over a restricted SSH account. Telchar centrally chooses the machine and transfers required paths.

The backend configuration should describe capabilities:

```text
name
address
systems
supported features
maximum jobs
administrative priority
```

A static SSH backend is useful for existing hardware, specialized machines, Darwin builders, or resources not managed by a cluster scheduler.

SSH must remain restricted to Nix build service behavior. The backend must not provide a general-purpose shell to Telchar credentials.

### Nomad batch backend

The Nomad backend creates one batch execution per derivation.

```text
Telchar request
      │
      ▼
Nomad batch job
      │
      ├── architecture and feature constraints
      ├── explicit CPU, memory, and disk resources where known
      ├── request-scoped credentials
      ├── input staging
      ├── one derivation realization
      ├── output collection
      └── terminal report
```

Nomad owns:

- Node selection.
- Resource placement.
- Pending allocation state.
- Allocation lifecycle.
- Runtime logs.
- Node draining.

An external autoscaler may inspect pending allocations and cluster metrics to adjust the underlying compute fleet. Telchar neither provisions nor identifies individual Nomad clients.

One-shot batch execution has an important lifecycle advantage: build lifetime and allocation lifetime are aligned. Normal node draining can stop new allocations, wait for running builds, and then remove capacity without inventing an SSH-session drain protocol.

The executor should receive a request ID or authenticated request descriptor, not a shell command assembled from user-controlled text.

### Future backends

Possible future backends include:

- Kubernetes Jobs.
- A cloud batch service.
- A dedicated worker protocol.
- A provider that reserves ephemeral machines.

These are not initial requirements. Their possibility validates the backend boundary; it does not justify implementing speculative abstractions.

## Input and output movement

Execution is only useful if every backend can obtain the exact derivation and input closure and return valid outputs.

The baseline transfer path is always through Telchar's gateway store:

```text
Client -> Telchar store -> executor
Executor -> Telchar store -> client
```

This path must work without a shared cache.

A backend may first substitute paths from its normal configured caches, then obtain any missing private paths from Telchar.

After execution:

1. The executor reports expected outputs.
2. Telchar imports them into its store.
3. Telchar verifies the result and expected path metadata.
4. Telchar returns a successful build result to the client.
5. Optional cache publication occurs according to policy.

Telchar must not report success merely because a backend process exited successfully. Required outputs must be present and valid at the gateway boundary.

## Optional binary-cache integration

A binary cache is an optimization layer, not a correctness dependency.

The cache may be used at three points:

```text
Client-side substitution
        │
        ▼
Telchar read-through substitution
        │
        ▼
Executor input substitution
        │
        ▼
Optional output publication
```

### Client-side cache

Clients may already use public or organizational substituters. A cache hit prevents the request from reaching Telchar. Telchar does not need to control this configuration.

### Gateway read-through cache

Before dispatch, Telchar may check configured substituters for expected outputs. This can avoid duplicate execution when another client or CI system has recently published the result.

Cache lookup must be bounded by a short timeout and fail open to normal execution. An unavailable optional cache must not make builds unavailable.

### Executor input substitution

Executors may use public or shared caches for common input paths. Missing paths must still be obtainable directly from Telchar.

This preserves support for:

- Local source trees.
- Unpublished derivations.
- Private inputs.
- Deployments without a shared cache.

### Output publication

Successful outputs may be published from Telchar to Attic or another cache.

Default publication should be asynchronous:

1. Import and verify outputs in the gateway store.
2. Return the result to the waiting client.
3. Queue cache publication.
4. Retry publication independently.

Deployments may later choose required synchronous publication for selected identities or workloads, but that should not be the default.

Telchar should centralize cache write credentials. Executors should normally have read access to shared caches and request-scoped access to Telchar inputs, but no organization-wide cache publication credential.

### Privacy policy

Telchar must not automatically publish every client-uploaded input path.

Inputs may contain:

- Proprietary source code.
- Local worktrees.
- Generated sources.
- Sensitive names or content.
- Material intended only for one build.

A safe default is:

```text
client-uploaded inputs: private to Telchar and selected executor
successful outputs: optionally publish according to explicit policy
```

Even outputs may contain proprietary material, so publication policy must remain configurable.

### Cache implementation boundary

Backend APIs must not mention Attic specifically. Cache lookup and publication are separate responsibilities from execution.

The first implementation may invoke stable external tools such as `nix copy` or `attic push` rather than linking to cache-specific internals. Credentials should use file-based service credentials or another secret-delivery mechanism, never plaintext main configuration.

## Identity and quotas

Source IP is useful audit context and a possible emergency fallback, but it is not a sufficient primary identity for organizational quotas.

Source IP is unstable or shared when users are behind:

- NAT.
- Corporate VPNs.
- Shared office networks.
- IPv6 privacy addressing.
- SSH bastions or proxies.

The preferred identity hierarchy is:

1. SSH public-key fingerprint.
2. OpenSSH certificate key ID and principals, when present.
3. Source IP as audit metadata and fallback classification.

A normalized requester record may contain:

```text
Requester
├── stable key fingerprint
├── certificate key ID
├── certificate principals
├── source IP
└── configured group or policy mapping
```

At work, each engineer or device should receive an individual key or certificate. A shared SSH principal can identify the service authorization while certificate key IDs and key fingerprints retain attribution.

Initial quotas should remain simple:

```text
global max queued
global max running
per-identity max queued
per-identity max running
```

Group quotas, weighted priorities, and reservations can be added when real usage demonstrates a need.

## Scheduling and fairness

A shared build service must prevent one client from consuming every available slot.

The first scheduler should support:

- Separate queues or accounting per requester identity.
- Round-robin selection among identities with runnable work.
- Per-identity running limits.
- Global running limits.
- Administrative priority classes where explicitly configured.
- Backend compatibility by system and features.

Potential later policies include:

- Team quotas.
- CI versus interactive priorities.
- Cost-aware backend preference.
- Affinity toward warm static builders.
- Derivation deduplication.
- Per-project limits.

These should follow observed requirements. Telchar should not begin as a research scheduler.

## Duplicate requests

Multiple clients may request the same derivation concurrently.

A future scheduler may coalesce equivalent requests into one execution and attach multiple interested clients. Correct equivalence must account for Nix semantics, build mode, and relevant options; it must not be guessed from a package name.

The first version may defer active deduplication. Store checks should still prevent unnecessary execution after one result has completed and entered the gateway store.

## Cancellation

Client disconnect and build cancellation are not always equivalent.

A disconnect may mean:

- The user cancelled.
- The network failed.
- SSH restarted.
- The client daemon restarted.
- Another interested client still wants the same result.

A conservative initial policy is:

- Cancel queued work that has no remaining attached requester.
- Allow already running work to finish and populate the gateway store.
- Record that the original request detached.
- Support explicit backend cancellation for administrative use and future reference-counted requests.

Immediate cancellation of every running build on transport loss wastes work and creates difficult result-transfer races.

## Failure classification and retries

Telchar should distinguish at least:

- Build failure: the derivation ran and failed.
- Infrastructure failure: allocation, machine, network, or executor failed.
- Admission failure: quota or policy rejected the request.
- Input failure: required paths could not be staged or verified.
- Output failure: outputs could not be collected or verified.
- Cancellation.
- Internal gateway failure.

Automatic retries should initially be limited to failures known to be infrastructure-related and safe to retry. A failed derivation must not be blindly retried as if a different machine will fix it.

Retries must be bounded and visible in execution history.

## Security model

Nix derivations are executable, potentially hostile code. Organizational deployments must treat build executors as an untrusted workload environment even when all engineers are trusted people.

### Gateway

The gateway should:

- Run with the minimum privileges needed for its Nix store and SSH service.
- Expose only the restricted Nix protocol endpoint.
- Validate store paths and protocol limits.
- Bound uploads, queues, logs, and request counts.
- Keep cache publication credentials away from executors.
- Record requester and backend attribution.
- Avoid shell interpolation.

### Executors

Executors should:

- Enable Nix sandboxing.
- Receive no production credentials.
- Have constrained network access.
- Be isolated from sensitive workloads.
- Use request-scoped credentials for gateway access.
- Have bounded CPU, memory, disk, and runtime where the backend supports them.
- Be disposable where practical.

A Nomad deployment should use a dedicated node pool or equivalently strong isolation. Scheduling arbitrary derivations beside secrets-bearing production workloads is unsafe.

### SSH

SSH access must prohibit:

- Interactive shells.
- PTYs.
- Agent forwarding.
- TCP forwarding.
- X11 forwarding.
- Arbitrary commands.

SSH host certificates and client certificates are supported deployment choices, not core project dependencies.

### Multi-tenancy

A first public release should describe its trust assumptions explicitly. Authentication and quotas do not alone provide safe hostile multi-tenancy. Store visibility, output publication, logs, backend isolation, and network policy all affect tenant separation.

## Autoscaling boundary

Telchar produces demand; infrastructure reacts to it.

For a Nomad backend:

```text
Telchar submits constrained batch job
        │
        ▼
Nomad places or leaves allocation pending
        │
        ▼
External autoscaler observes pending demand
        │
        ▼
Infrastructure adds eligible clients
        │
        ▼
Nomad runs allocation
```

Scale-down is likewise external:

```text
External autoscaler selects idle capacity
        │
        ▼
Scheduler drains node
        │
        ▼
No new batch allocations are placed
        │
        ▼
Existing allocations finish
        │
        ▼
Infrastructure removes node
```

One derivation per batch allocation gives the infrastructure scheduler a concrete unit to place, observe, drain, and terminate. Telchar should expose queue and execution metrics but must not embed provider-specific scaling policy.

## Observability

Telchar should provide structured logs and Prometheus-compatible metrics.

Useful metrics include:

- Accepted, rejected, queued, running, completed, and failed requests.
- Queue wait duration.
- Execution duration.
- Input staging and output collection duration.
- Bytes transferred by direction.
- Cache hit, miss, timeout, and publication outcomes.
- Running requests by backend, system, and feature class.
- Backend submission and infrastructure failures.
- Per-identity quota rejections without exposing high-cardinality raw identity labels.
- Detached and cancelled requests.
- Retry counts and reasons.

Every request should have a stable request ID propagated into:

- Gateway logs.
- Scheduler state.
- Backend execution metadata.
- Nomad job or allocation metadata.
- SSH execution logs.
- Cache publication work.

An administrative CLI or API should eventually support:

```text
telchar status
telchar queue
telchar jobs
telchar job show <id>
telchar job cancel <id>
telchar backends
telchar backend drain <name>
```

The exact command surface should follow implementation needs.

## Configuration direction

Telchar should use a human-readable configuration format such as TOML.

A possible shape, for illustration only:

```toml
[server]
listen = "0.0.0.0:2222"

[quota.defaults]
max_queued = 20
max_running = 4

[[backends]]
name = "local"
kind = "local"
systems = ["x86_64-linux"]
max_jobs = 2
priority = 10

[[backends]]
name = "arm-workers"
kind = "ssh"
systems = ["aarch64-linux"]
max_jobs = 8
priority = 50

[[backends]]
name = "nomad-linux"
kind = "nomad"
systems = ["x86_64-linux", "aarch64-linux"]
priority = 40

[cache]
substituters = [
  "https://cache.nixos.org"
]
lookup_timeout = "3s"

[cache.publisher]
enabled = false
mode = "async"
```

Secrets must be referenced by file or injected through service credentials, not embedded in this file.

Configuration should be validated at startup. Unknown backend kinds, impossible capacities, duplicate names, and unsupported systems should fail with actionable errors.

## Rust implementation

Rust is the preferred implementation language.

It is a good fit for:

- Binary protocol correctness.
- Async SSH and network streams.
- Backpressure-aware NAR transfer.
- Strong internal state models.
- Backend interfaces.
- Long-running daemon reliability.
- Static or low-dependency deployment artifacts.
- Property and fuzz testing of protocol parsers.

A likely async runtime is Tokio. Exact libraries should be selected after focused prototypes, especially for:

- SSH server behavior.
- Nix worker-protocol reuse.
- NAR parsing and streaming.
- Nix store interaction.
- Nomad API access.
- Durable state.

The project should avoid committing to a broad framework before proving the worker-protocol and store vertical slice.

## Suggested internal modules

Initial source organization could resemble:

```text
src/
├── main.rs
├── config.rs
├── identity.rs
├── protocol/
├── store/
├── admission/
├── scheduler/
├── execution/
├── backend/
│   ├── local.rs
│   ├── ssh.rs
│   └── nomad.rs
├── cache/
├── metrics.rs
└── state.rs
```

This is a direction, not a required crate split. Begin as one crate unless compile boundaries or reuse justify a workspace.

## Delivery phases

### Phase 0: protocol spike

Goal: prove a stock Nix client can communicate with a Rust endpoint.

- Start restricted SSH endpoint or invoke stdio server directly in tests.
- Complete Nix worker-protocol handshake.
- Exercise protocol against real Nix.
- Identify the minimum required operation set for remote-builder mode.
- Extract the useful `rio-nix` protocol code into the Telchar repository with BSD-3-Clause attribution.
- Remove unrelated functionality and identify any protocol gaps through compatibility tests.

No scheduler, Nomad, quotas, or cache publication yet.

### Phase 1: local vertical slice

Goal: complete a real derivation through Telchar.

- Gateway-local Nix store.
- Local backend.
- Real input import.
- One derivation execution.
- Log forwarding.
- Output verification and return.
- Real NixOS VM integration test.

Success criterion:

```text
stock Nix client -> Telchar -> local backend -> output returned
```

### Phase 2: admission and durable execution state

- Stable requester identity.
- Global and per-identity limits.
- Fair queue.
- Durable request records.
- Administrative status and cancellation.
- Structured metrics.

### Phase 3: static SSH backend

- Capability configuration.
- Input staging.
- Restricted SSH execution.
- Output collection.
- Infrastructure/build failure distinction.
- End-to-end test with a real SSH builder VM.

### Phase 4: Nomad batch backend

- One batch job per derivation.
- System and feature constraints.
- Request-scoped executor credentials.
- Pending/running/completed reconciliation.
- Allocation log integration.
- Cancellation.
- End-to-end test against a local Nomad development cluster or isolated test environment.

### Phase 5: optional cache integration

- Read-through substituters.
- Executor substitution.
- Direct Telchar fallback for missing private inputs.
- Asynchronous output publication.
- Cache outage tests proving builds remain functional.

### Phase 6: organizational hardening

- Group policy.
- Priority classes.
- Stronger isolation guidance.
- Audit exports.
- High-availability state if required.
- Duplicate request coalescing if justified.
- Broader compatibility and load testing.

## Testing strategy

Telchar should use test-driven development for protocol features and bug fixes.

### Unit tests

- Identity normalization.
- Quota decisions.
- Fair queue behavior.
- Backend capability filtering.
- Retry classification.
- Configuration validation.
- Protocol framing and version handling.

### Property and fuzz tests

Protocol and NAR handling are suitable for:

- Truncated input.
- Oversized lengths.
- Unknown operation codes.
- Invalid store paths.
- Malformed structured errors.
- Arbitrary byte streams.
- Version-boundary behavior.

### Integration tests

Use real components rather than mocks for core correctness:

- Real stock Nix client.
- Real Nix stores.
- Real SSH transport where SSH behavior is under test.
- Real local execution.
- Real remote SSH builder.
- Real Nomad development agent for Nomad backend tests.
- Optional real binary-cache fixture where practical.

The primary acceptance test should prove that the client cannot build locally, submits work to Telchar, and receives a verified output produced by the selected backend.

Expected error logs should be captured and asserted so passing test output remains clean.

## Inspirations and related work

Telchar builds on lessons from existing Nix build systems rather than pretending the problem is unexplored.

### Nix remote builders

Nix's worker protocol and `ssh-ng://` transport provide the compatibility contract. Telchar deliberately preserves this client experience while centralizing scheduling behind one endpoint.

### rio-build

[rio-build](https://github.com/lovesegfault/rio-build) is the strongest architectural inspiration.

Relevant ideas include:

- Implementing the standard Nix worker protocol at a gateway.
- Keeping clients unmodified.
- Treating each derivation as a schedulable execution unit.
- Using one-shot executors.
- Separating gateway, scheduler, store, and executor responsibilities.
- Central scheduling and observability.
- Drain-aware lifecycle management.

Rio-build is Kubernetes-native and includes a sophisticated DAG scheduler, chunked CAS, FUSE-backed stores, controller, and autoscaling design. Telchar intentionally starts with a smaller scope:

- Standard remote-builder mode rather than full remote-store DAG scheduling.
- A normal gateway Nix store rather than a custom CAS.
- Pluggable backends rather than a Kubernetes-only executor controller.
- External infrastructure autoscaling.

Rio-build is archived and has limited adoption, so Telchar will not depend on it as an active upstream project. Its `rio-nix` crate is deliberately isolated from other `rio-*` crates and includes worker-protocol, ATerm, NAR, store-path, build-result, and structured-error parsing plus fuzz targets. Telchar will selectively import useful `rio-nix` code into its own repository under rio-build's BSD 3-Clause license, preserve the required copyright and license notices, record the source revision, and maintain the resulting code directly. Reimplementation remains appropriate for pieces that do not fit Telchar, but duplicating already-tested binary protocol machinery provides no benefit.

### Yensid

[Yensid](https://github.com/garnix-io/yensid) demonstrates that a stable SSH endpoint can hide a pool of Nix builders and improve global load balancing without client changes.

Relevant lessons:

- One stable remote-builder endpoint.
- Layer-4 transparency.
- Central backend choice.
- SSH certificate support for interchangeable hosts.

Yensid uses HAProxy and static backend configuration, with custom Lua as an extension seam. Telchar moves scheduling above the TCP-connection layer so it can apply quotas, understand derivation lifecycle, and dispatch one-shot jobs.

### Trébuchet

[Trébuchet](https://github.com/Mic92/tribuchet) demonstrates a central hub with dynamic workers, capability matching, draining, and remote execution. It also highlights the operational benefits of workers dialing a scheduler and of separating build execution from classic SSH builder topology.

Trébuchet currently relies on Nix's experimental external-builders feature, while Telchar prioritizes compatibility with stock remote-builder clients.

### Guardian Project nix-builder-autoscaler

The Guardian Project's `nix-builder-autoscaler` demonstrates explicit slot reservation, EC2 lifecycle states, readiness, draining, and termination for elastic Nix builders.

Its Buildbot, Tailscale, and HAProxy integration is not Telchar's architecture, but its state-machine treatment of infrastructure failures and draining is useful prior art.

### Hydra and hydra-provisioner

Hydra and `hydra-provisioner` demonstrate queue-driven build-machine provisioning by system and required features. Telchar does not adopt Hydra's CI/jobset model, but retains the lesson that scaling signals should come from queued build demand rather than raw proxy connection counts.

## Open questions

The following require prototypes or explicit project decisions:

1. Which portions of the imported `rio-nix` code fit Telchar unchanged, and which require replacement?
2. What is the minimum operation set required across supported Nix versions?
3. Should the first SSH endpoint use an embedded Rust SSH server or OpenSSH with a forced command?
4. Should the first gateway use the system Nix store or an isolated store root?
5. How should executors obtain request-scoped access to private input paths?
6. What backend result format preserves all required Nix `BuildResult` semantics?
7. Which failures are provably safe to retry?
8. How should log streaming resume after gateway or backend reconnects?
9. What Nix and Lix version range will the project support initially?
10. When does SQLite cease to be sufficient for durable state?
11. What is the minimum safe isolation contract for a shared Nomad executor pool?
12. Should cache publication use `nix copy`, an Attic CLI, or a native API first?

These are design questions, not reasons to expand the initial scope. The first vertical slice should answer the protocol and store questions before work begins on backend breadth.

## Initial project charter

> Telchar accepts standard Nix remote-build requests over SSH and schedules each derivation onto a compatible execution backend. It owns protocol compatibility, admission, queueing, quotas, input and output transfer, execution tracking, and result reporting. Backends own execution. Caches are optional optimizations. Machine provisioning and autoscaling remain external concerns.
