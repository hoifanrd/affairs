---
authors: hoifanrd
state: prediscussion
discussion:
labels: infrastructure, security
---

# [RFD] Interactive Code-Execution Sandbox

## Overview

This RFD proposes the design of a `sandbox` extension that executes student-submitted code interactively on behalf of the ZINC core platform. Where the grading pipeline (a batch system) accepts a one-time submission, runs it through a graded workflow, and returns a score, the sandbox accepts a code snippet, runs it inside a strongly isolated container, and returns standard output, standard error, exit code, peak memory, and timing within a latency budget that supports an interactive "run my code" button in the student client.

The design centres on a **warm pool of long-lived, security-hardened containers** rather than spawning a fresh container per submission. The remainder of this document justifies the warm-pool decision against an empirical benchmark, specifies the per-container security model, the inter-job cleanup contract, and the failure-recovery model, and lays out the execution-backend abstraction so that additional backends (Kubernetes, Nomad) can land in follow-up RFDs without reshaping the surrounding architecture.

## Background

The grading pipeline ([RFD 0009](../0009/README.md) establishes the extension boundary; the pipeline is its own extension) is engineered for *correctness under load*: each submission gets its own container, executed by Concourse, with multi-stage isolation between compile, test, and score phases. The trade-off is latency — a fresh container plus image pull plus Concourse step machinery puts cold-path latency in the seconds-to-tens-of-seconds range, which is acceptable for a "submit assignment" flow but not for an "let me try my code now" flow.

The interactive use case has different requirements:

- **Sub-second latency** from click to first character of output. Students treat the button like a REPL.
- **Concurrent users** clicking simultaneously during a tutorial or lab. A class of 200 students all attempting to run code at the start of an exam is a hot path.
- **Strong isolation** between students. A student must not be able to read, signal, or otherwise observe another student's code, output, or processes, even though their containers may sit on the same host kernel.
- **Bounded blast radius** for misbehaving submissions. A fork bomb, a memory bomb, an infinite output loop, or a malicious syscall sequence must not destabilise the host or the next student's run.
- **Per-question resource policy** controllable by the instructor. Memory, CPU, and timeout vary by question difficulty.

None of these are met by the pipeline's per-run dispatch model. A dedicated extension with its own execution strategy is justified.

### Why not reuse the pipeline runner

The pipeline's container-per-run model is engineered around Concourse, which sets up resource volumes, mounts grader inputs, executes scripted stages, and tears down. Even with a hot Docker daemon and a pinned image, the create + start + first-exec + stop + remove cycle is dominated by daemon-side overhead. We measured this directly (see [Empirical justification of the warm pool](#empirical-justification-of-the-warm-pool-docker-backend) below) and observed a median full-cycle latency of roughly **254 ms** on commodity hardware, with the bare `Exec` itself accounting for only **39 ms** of that. The 6× overhead is not amortisable across a single interactive submission, which is the unit of latency the user perceives.

### Why not "just run the code on the host"

The submission is untrusted code, written by hundreds of students per course, sometimes intentionally exploring the boundary of what the platform will allow. The minimum safe substrate is a Linux container with restricted syscalls, restricted capabilities, restricted filesystem, and no network. The sandbox extension is the layer that codifies that "minimum safe substrate" so the rest of the platform can treat code execution as a black box that takes bytes in and returns bytes out.

## Scope

### In scope

- A warm-pool execution model for interactive code submissions, keyed by `(source, sourceID)`.
- The per-executor security configuration (filesystem, namespaces, seccomp, AppArmor, capabilities, resource limits, ghost-helper isolation).
- The inter-job reset contract and its verified post-conditions.
- The failure-recovery model (event listener, heartbeat staleness, dirty release).
- The Docker execution backend.
- The job tracker (shared-cache-backed) and the HTTP/SSE surface that drives the "run my code" flow from the student client.
- The linter as a peer system in the same extension.

### Out of scope (deferred to follow-up RFDs)

- **Additional execution backends.** Kubernetes and Nomad are out of scope here; the `Backend` interface is described only as the target abstraction shape (see [Execution backend](#execution-backend)). A follow-up RFD will land each backend.
- **Operator autoscaling beyond today's behaviour.** A configuration hook is described for future use; no `executor.policies` block is consumed by the implementation today.
- **Cross-replica scheduling.** Multi-replica deployments are addressed by the upstream load balancer's consistent-hash routing on `(source, sourceID)`; no app-level pool ownership is modelled here.
- **Per-language toolchain images.** The image reference is part of the pool configuration; image authorship is the operator's concern.
- **Remote-backend stdio capture and metrics exposure** via in-container HTTP. Deferred to the per-backend follow-up RFDs.

## Proposal

The sandbox extension owns four responsibilities:

1. **A pool of warm executor containers** keyed by `(source, sourceID)`. For the examination use case, `sourceID` is the document identifier. Each pool is sized by the instructor's per-document configuration (executor count, memory, CPU, timeout). Pools are created lazily on first submission and torn down when idle past a threshold.
2. **A per-executor security configuration** that layers seccomp, AppArmor, capability drop, read-only root filesystem, scoped tmpfs mounts, no network, and resource limits. An in-container helper binary (`ghost`) wraps each untrusted command with additional Landlock and network-namespace isolation. The same helper writes stdout, stderr, and a liveness heartbeat to a per-executor scratch directory.
3. **An inter-job reset contract** that verifies — does not merely request — that a container returns to a known-clean state between two students' jobs. Reset failures route the container to destruction, not reuse.
4. **An execution backend abstraction** behind which the implementation runs. The first backend is Docker; the abstraction is shaped so that future backends (Kubernetes, Nomad) can be added without reshaping callers.

The remainder of this RFD specifies each responsibility in detail.

## Empirical justification of the warm pool (Docker backend)

The warm-pool decision is the most consequential single choice in this design — it shapes the security model, the cleanup contract, and the failure model. To justify it we measured the actual cost of the two alternatives against the production code path.

**Scope of this section.** The measurements below were collected against the Docker backend running on a single Linux host (host-mode docker socket, container runtime co-located with the extension). They establish a *lower bound* on cold-cycle overhead — the most favourable setting for per-job dispatch. They are not representative of distributed backends, whose control-plane round-trips amplify the gap (see [Why Docker-host numbers understate the case for warm pooling](#why-docker-host-numbers-understate-the-case-for-warm-pooling) below). The figures below should not be quoted as K8s or Nomad latency; the per-backend follow-up RFDs re-run this benchmark methodology against each backend's primitives.

### Method

A benchmark binary exercises the executor package directly with the same security configuration as the production code: inline seccomp profile, AppArmor `docker-default`, `CapDrop: ALL`, read-only rootfs, tmpfs `/tmp` and `/workspace`, `NetworkMode: none`, no swap, private cgroup namespace, ghost-wrapped command. The same trivial workload (`echo hello`) runs in both modes so the only variable is sandbox overhead.

- **Warm mode**: one container is created once. The benchmark times `N` calls to `Exec` on that container.
- **Cold mode**: each of `N` iterations performs the full lifecycle `NewExecutor + Start + Exec + Stop + Remove`.

Both modes use the same memory limit (256 MiB), CPU limit (1.0), workload command, and image (`core/zinc-sandbox:local`, ~162 MiB). The image is pre-loaded into the local Docker daemon for both modes, so neither mode pays image-pull cost — what is being measured is daemon-side container lifecycle overhead, not registry latency.

### Results (Docker host, 30 iterations, commodity Linux host with cgroup v1)

| Mode | Min | p50 | p95 | p99 | Max | Mean |
|---|---|---|---|---|---|---|
| Warm — reuse pooled container | 38 ms | **39 ms** | 42 ms | 43 ms | 43 ms | 40 ms |
| Cold — full create+start+exec+stop+remove cycle | 238 ms | **254 ms** | 271 ms | 276 ms | 276 ms | 255 ms |

- **Median ratio**: cold is **6.4× warm**.
- **Absolute per-job overhead saved by the warm pool**: ~214 ms.

These figures are specific to a single-host Docker deployment with the container runtime co-located with the extension. They are not representative of, and should not be extrapolated to, remote Docker daemons, Kubernetes, or Nomad.

### Implications (Docker host)

- **Per-user latency.** 39 ms is sub-perceptual ("instant"); 254 ms is consciously noticeable and crosses the threshold where students start clicking the button twice. The warm pool keeps the interactive flow inside its budget; per-run dispatch does not.
- **Host throughput under load.** A single sandbox replica running back-to-back submissions can sustain ~25 warm jobs/second per slot or ~4 cold jobs/second per slot. The 6.4× factor multiplies through pool size: a 5-slot warm pool absorbs ~125 jobs/second, whereas the same 5 slots running cold-cycle would cap at ~20 jobs/second.
- **Image pull dominance is not the story.** Both modes find the image locally in the daemon. The 214 ms gap is daemon-side state setup, AppArmor profile apply, seccomp profile installation, cgroup creation, network namespace creation, and the tmpfs mounts — none of which the warm pool pays per job.

### Why Docker-host numbers understate the case for warm pooling

A reader might reasonably argue that 254 ms is borderline tolerable — uncomfortable but not disqualifying — and that the warm pool's complexity (Reset, dirty release, state machine) may be over-engineering for a 6× win on a single host. The reason this argument doesn't survive contact with the deployment targets is that **Docker-host cold-cycle is the floor, not a representative point.** On the distributed backends the design is shaped to support, the cold-cycle balloons while the warm-cycle stays roughly flat.

Concretely:

- **Kubernetes.** The full Pod cold cycle — apiserver admission, scheduler bind, kubelet poll-and-pickup, sandbox-pod (pause container) create, container create + start, readiness, then `kubectl exec` through the apiserver proxy, then graceful delete — is in the **seconds** range under typical conditions. Even with the image cached on the node, a one-second cold cycle is the optimistic case; under load or with a non-trivial scheduler workload, multi-second cold cycles are routine. The warm path, by contrast, is just one `apiserver → kubelet → container` exec round-trip against an already-running Pod — typically tens of milliseconds.
- **Nomad.** Allocations require a Raft commit on the server, then a client agent pickup, then sandbox setup before `nomad alloc exec` becomes available. The cold cycle is similar in shape to K8s — comfortably seconds — while the warm exec is again one round-trip.

The architectural conclusion is therefore not "warm pool is 6× faster on Docker" — it is "warm pool keeps interactive submissions inside their latency budget on every backend we intend to support, while per-job dispatch fails the budget by orders of magnitude on K8s and Nomad." Docker-host is the *most favourable* setting for cold-cycle dispatch, and it already loses by 6×; on the distributed backends the gap is qualitatively wider, with the cold cycle pushed from sub-second to multi-second territory. Each per-backend follow-up RFD (see [Other backends](#other-backends)) re-runs the same benchmark against its own primitives to put concrete numbers on this.

The cost the warm pool pays for these wins is not free: shared infrastructure has to be wiped between users, which is the subject of [Inter-job reset contract](#inter-job-reset-contract) below. The architectural shape of the design (warm pool, security layers, reset contract, backend interface) is chosen so the same conclusion holds — and grows — on other backends.

## Architecture

### Component overview

```
sandbox extension
├── pool manager (one Pool per (source, sourceID))
│   ├── lifecycle: create, idle-evict, hot-config-update, shutdown
│   ├── slot: primary state (creating | idle | busy)
│   │         + flags (dead, pendingRecreate)
│   └── failure detection: backend event listener + heartbeat + dirty-release
├── executor (one container per slot)
│   ├── security configuration (seccomp, AppArmor, capabilities, etc.)
│   ├── ghost helper baked into the image (heartbeat + per-exec wrapping)
│   ├── output capture via host bind-mount
│   └── reset contract with post-condition verification
├── linter (singleton container for syntactic feedback; out of scope of warm-pool)
└── job tracker (shared-cache-backed; one record per submission)
```

### Pool lifecycle

A pool is identified by `(source, sourceID)`. Today only one source is registered (`examination`, keyed by document ID); the abstraction is open so that other sources can register their own configuration extractors.

A slot has a **primary state** (one of three) plus two **orthogonal modifier flags**. The primary state describes where the slot is in the acquire/release cycle; the modifiers record sticky conditions discovered asynchronously that route the slot to destruction on release.

| Primary state | Meaning | Created by | Cleared by |
|---|---|---|---|
| `creating` | Slot reserved under the pool mutex before any container work | Acquire / Prewarm | Successful create OR rollback on error |
| `idle` | Container started, helper heartbeat alive, ready for use | Successful start OR successful Reset | Acquire |
| `busy` | Slot handed to a job | Acquire | Release |

| Modifier flag | Meaning | Set by | Acted on by |
|---|---|---|---|
| `dead` | Backend reported `die` / `oom` event for this container | Event listener callback | Acquire-time skip + background cleanup; Release-time destroy |
| `pendingRecreate` | Hot config update marked this slot for replacement on next release, OR caller released with `dirty=true` because Reset failed or could not be verified | Config update path; Release(dirty=true) | Release-time destroy + replacement provisioning |

`dirty` is not a stored slot state — it is a per-release argument passed to `ReleaseExecutor` that, when true, sets `pendingRecreate` and routes the slot to destruction. A slot can carry either modifier flag while in any primary state; both flags are checked at release time to decide between "return to `idle`" and "destroy and replace."

Pools are created lazily by the first `Acquire` for a `(source, sourceID)`. The instructor's per-document configuration (image reference, executor count, memory/CPU limits, timeout, default command, run-attempt cap) is fetched at pool creation. The pool maintains an executor sequence counter that produces stable, human-readable container names; sequence allocation is performed in shared cache so multiple sandbox replicas cannot produce colliding names.

### Acquire path

A job entering the system performs the following sequence under the per-pool mutex:

1. Scan idle slots. Skip any whose `dead` flag was set by the event listener. Skip any whose helper heartbeat is stale (more than ~30 seconds old). Acquire the first surviving idle slot.
2. If no idle slot is available and the pool is below its executor limit, reserve a `creating` slot under the lock, then perform the container create and start *outside the lock*, finally installing the started container into the slot under the lock again.
3. If the pool is at its executor limit, append a buffered waiter channel to the pool's waiter queue and block. The waiter is woken by a future `Release` or by the caller's context cancellation.

The "scan idle slots and skip dead/stale ones" step is a hot path; it must not perform Docker calls. Both signals (`dead` flag, helper heartbeat) are designed so that the check is a single memory read or a single filesystem read of a small file. The reasoning is that the steady-state acquire latency must not be paced by daemon round-trips.

### Release path

On release, the caller passes a `dirty` flag. The flag is `true` when the inter-job Reset failed or could not be verified — when the container *might* still hold a previous job's process or file. The release performs:

1. If the slot is `pendingRecreate` (hot config update) OR `dead` (event listener saw a death) OR the caller's `dirty` flag is set: the container is destroyed in a background goroutine tracked by the pool's wait group, and if there are queued waiters and the pool is not closed, a *replacement* container is created for the head waiter. This is the only way a queued waiter can be unblocked when the previous holder's slot was destroyed, because the lazy-create branch on the acquire path only fires for *new* acquire calls.
2. Otherwise, mark the slot `idle`. If there is a queued waiter, hand the slot to that waiter directly (transitioning back to `busy`).

The `bgWg` tracks both teardowns and replacement creations; pool removal waits on it before deleting the pool from the manager so a recreated pool with the same key cannot collide with orphan teardown.

### Concurrency rule

Exactly one mutex per pool, held briefly, **never across blocking I/O** (Docker, network, database). Image pulls during hot config updates are performed in the gap between two `Lock` / `Unlock` cycles. Background work scheduled while holding the mutex always increments the pool's wait group before unlocking, so a concurrent shutdown's `Wait` cannot miss it. A queued waiter that cancels its own context drains its channel under the mutex and, if a sender granted it an executor in the race, releases that executor back clean.

## Per-executor security model

The threat model is: a student's submission is arbitrary code from an untrusted author, with full read access to its own writable filesystem and CPU. The trusted boundary is the host kernel. The submission must not be able to escape its container, exhaust host resources, observe sibling containers, persist state between submissions, or communicate over the network.

The defence is **layered** — six independent mechanisms must all be defeated for an escape, and each mechanism has a documented failure mode the others mitigate.

### Layer 1 — Filesystem

- **Read-only root filesystem.** The container image is mounted read-only at `/`. Writes are confined to mounts described below.
- **`/tmp`**: tmpfs, ~64 MiB, `noexec, nosuid`.
- **`/workspace`**: tmpfs, ~32 MiB, `exec, nosuid`. This is the only writable filesystem location where the student can run a compiled binary.
- **`/output`**: bind mount to a per-executor host scratch directory. The container writes stdout / stderr / heartbeat files here; the host reads them directly. The host directory is created with mode `0711` and the three output files with mode `0666` so the container does not require `CAP_DAC_OVERRIDE` to write them. The unwritable directory mode prevents a compromised container from replacing the output files with symlinks pointing at arbitrary host paths.
- `XDG_*` and `HOME` are pointed at `/tmp` so libraries that try to cache to `$HOME` (matplotlib font cache, Java preferences, etc.) do not crash against the read-only root.

### Layer 2 — Process and network namespaces

- `NetworkMode: none`. No interface inside the container, no route to the host or other containers.
- `CapDrop: ALL`. Every Linux capability is dropped — most importantly `CAP_NET_*`, `CAP_SYS_ADMIN`, and `CAP_KILL`.
- Container runs as a **per-executor unique UID** allocated from a shared sequence counter starting at `100000`. Allocating UIDs from a cluster-shared counter rather than process-local prevents two sandbox replicas on the same host from colliding on UID and accidentally sharing the kernel's per-UID resource accounting (notably `RLIMIT_NPROC`).

### Layer 3 — Seccomp

A custom seccomp profile attached to every container. The profile is an **allowlist** — anything not on the list returns `ENOSYS` (errno 38) rather than killing the process with `SIGSYS`. Returning `ENOSYS` lets glibc gracefully degrade newer syscalls (`clone3 → clone`, `openat2 → openat`) without crashing student code on a modern kernel. The authoritative list of allowed syscalls lives alongside the implementation; this section commits only to the capability boundaries the profile enforces.

The capability boundaries the profile enforces:

- **No new-namespace creation by student code.** `clone` is argument-filtered to reject any `CLONE_NEW*` flag, and `unshare` is argument-filtered to reject `CLONE_NEWUSER`. The well-known unprivileged path into the kernel's user-namespace attack surface — `clone(CLONE_NEWUSER, …)` and `unshare(CLONE_NEWUSER)` — falls to `ENOSYS`. The helper's `unshare(CLONE_NEWNET)` for in-container network isolation deliberately remains reachable (and is then rejected by the kernel with `EPERM` because `CAP_SYS_ADMIN` is dropped; the helper silently ignores that).
- **No fallback to an unfiltered process-creation path.** `clone3` is excluded. Glibc treats `ENOSYS` as "kernel too old" and falls back to `clone`, which is argument-filtered.
- **No IPC-namespace state that can leak between students.** SysV IPC (`shmget`/`semget`/`msgget` and their op/ctl/at/dt counterparts, plus the i386 `ipc()` multiplexer) and POSIX message queues (`mq_*`) are excluded. These objects live in the container's IPC namespace, survive process death, and are not on any filesystem — so Reset cannot enumerate them, and a single object left behind by one student would leak into the next student's job. The block is at the syscall layer because that is the only place leak-freedom can be guaranteed. POSIX shared memory (`shm_open` + `mmap`) and POSIX semaphores (`sem_open`) are unaffected because they live on `/dev/shm`, which Reset wipes explicitly.

The principle the allowlist enforces is the same one Reset enforces from the other side: **anything Reset cannot reach is made unreachable at the syscall layer**. The two layers are paired by construction.

### Layer 4 — AppArmor (optional)

When the host supports AppArmor, the container is started with `apparmor=docker-default`. Detection is a filesystem check (`/sys/kernel/security/apparmor`). On hosts that report support but reject the profile at container-create time (some kernel/image combinations), the sandbox retries once without the AppArmor option and logs a downgrade warning. AppArmor is treated as an additive layer; the seccomp profile is the load-bearing protection.

### Layer 5 — Resource limits

- **Memory limit** with `MemorySwap == Memory`. This disables swap inside the container. Without it, Docker defaults `MemorySwap` to `2 × Memory`, which allows OOM-bound workloads to spill into swap on hosts that have swap enabled — which breaks OOM detection at the cgroup layer.
- **CPU limit** as a fractional CPU count, translated to nanoCPUs.
- **PIDs limit**: ghost's `RLIMIT_NPROC` caps student processes; the container's cgroup `pids` limit is set slightly higher (RLIMIT plus four reserved slots) so that management execs (`kill -9 -1`, `find -delete`) during Reset are not themselves blocked by a previously-fork-bombed pool.
- **Output cap**: a per-`Exec` goroutine polls the on-disk size of `/output/stdout` and `/output/stderr` every 50 ms. If their total exceeds 1 MiB while ghost-wrapped student code is running, the goroutine kills the entire container. Closing the Docker exec stream is insufficient because the student process holds an open file descriptor into the host bind-mounted file and keeps writing directly to the host disk — only killing the container revokes the descriptor. The dead container is observed by the event listener and replaced by the pool.

### Layer 6 — In-container ghost helper

A small statically-linked binary baked into the sandbox image at `/usr/local/bin/ghost`. It runs in three modes:

- **`heartbeat`**: the container's PID 1. Writes a Unix timestamp to `/output/.heartbeat` every ~10 seconds. The host's `IsHealthy` check reads this file directly, without any backend API call.
- **`run --exec --sandbox`**: wraps every *untrusted* command — that is, every command where the caller's `User != "root"` and `Stdin == nil`. Inside the container it installs Landlock rules, runs `unshare(CLONE_NEWNET)` for network isolation, sets `RLIMIT_NPROC`, and `dup3`s stdout and stderr onto pre-opened file descriptors pointing at `/output/stdout` and `/output/stderr`. Then it `execve`s the student command — no fork, no shell.
- (Future) **Object-storage upload mode and local metrics HTTP endpoint** for remote backends — covered by the per-backend follow-up RFDs (see [Other backends](#other-backends)).

Internal operations (`Reset`'s `kill -9 -1`, `WriteFile`'s `tee`) bypass ghost — they need stdin, run as root, and are not exposed to student input.

The combination of host-side container isolation and in-container ghost isolation gives two independent boundaries: even if a student found a way to escape the seccomp filter for a particular syscall, they would still face Landlock and the network namespace.

## Output capture

Stdout, stderr, and the helper heartbeat are written to a per-executor host scratch directory bind-mounted at `/output` inside the container. The host reads stdout and stderr by `open(..., O_RDONLY|O_NOFOLLOW)` on the corresponding files, with a 256 KiB read cap. When the on-disk size exceeds the cap, the truncated tail is replaced with an explicit marker so the student sees why output stopped rather than debugging silent loss.

The bind-mount path was chosen over capturing the Docker exec attach stream for two reasons:

- **Output survives crashes.** If ghost is OOM-killed mid-run, the bytes already flushed to the file are recoverable; whatever the exec stream had not yet delivered is gone.
- **Heartbeat is a host filesystem read.** The pool's idle-slot health check during the acquire hot path needs to be cheap; a daemon round-trip would dominate.

The output is **truncated**, not removed, between commands and between jobs. The truncation is best-effort within a single job (failure leaves stale bytes in the file for one more command, no security impact); on the inter-job boundary it is part of the verified Reset.

The cost of this choice is that the bind-mount is exactly the thing that does not work on backends where the host filesystem and the container's filesystem live on different machines. The per-backend follow-up RFDs address how output capture is re-implemented in that regime — see [Other backends](#other-backends).

## Inter-job reset contract

A warm container that has just finished a student's job must be brought back to a state observationally indistinguishable from a freshly created one before the next student's job can use it. The cost of getting this wrong is direct: state leakage between students.

Reset is a **verified post-condition**, not a sequence of best-effort commands. Each step is followed by a probe that measures the state of the container itself; only when the probes succeed does the slot return to `idle`.

### Sequence

1. **Kill lingering processes**. Inside the container, run `kill -9 -1` *as the executor's UID* (not as root — recall that `CAP_KILL` is dropped, so even root inside the container cannot signal processes owned by other UIDs). Every process in the container's PID namespace runs as the same UID by design, so this is sufficient. The command is retried with a short backoff between attempts if the PID table is temporarily exhausted by a fork bomb still being reaped. If retries are exhausted, Reset fails.
2. **Verify the kill**. As root inside the container, walk `/proc/[0-9]*/status` and treat any non-PID-1, non-self, non-zombie process as evidence of a failed kill. The shell script that performs this walk uses only builtins so its process tree is limited to a single `sh`; PID 1 is ghost. If anything else is alive, Reset fails. The verification protects against a future code change that introduces a process running as a different UID — in that case the unprivileged kill would silently miss it, and only the verification would catch the leak.
3. **Wipe filesystem state**. As root inside the container, run `find /workspace /tmp /dev/shm -mindepth 1 -delete`. `/dev/shm` is included because Docker mounts it as tmpfs regardless of the read-only root filesystem; it is not under `/workspace` or `/tmp`, so it would otherwise leak.
4. **Verify the wipe**. Run a verification script with two probes, each with an exact-zero baseline:
   - No process holds an open file descriptor into a wiped directory. `find -delete` removes the directory entry, but if a process holds an open fd the blocks stay allocated — `find` reports success while the space is pinned. The fd scan runs before the emptiness scan so the verifier's own `find` cannot show up as an open fd into the wiped directories.
   - No directory entries remain. Catches a `find -delete` that left something behind for any reason while still exiting `0` for what it reached.
5. **Truncate output buffers** on the host side. Best-effort within Reset; the writer for the next job is ghost, which `O_TRUNC`s on open anyway.

If any of steps 1–4 fail, Reset returns an error; the caller marks the release `dirty`; the pool destroys the container and provisions a fresh one. The next student does not get this container.

### What Reset does not enumerate, and how that is handled

Several kinds of state are not reachable by Reset, either because they have no filesystem representation (SysV IPC, POSIX message queues) or because they are kernel-namespace-private and survive process death. Anything in this class is **blocked at the security layer** instead — see the seccomp section. The principle: anything Reset can't reach is made unreachable.

## Failure recovery

A container becomes unsafe through one of three independent paths. Each has its own signal, and all three converge on the same action: destroy the container and provision a replacement.

| Signal | Source | Detection latency |
|---|---|---|
| Backend reported container death (`die` / `oom`) | Backend's event stream | Real-time, asynchronous |
| Helper heartbeat is stale (older than ~30 seconds) | Filesystem read during acquire | First acquire after staleness |
| Inter-job Reset failed or could not be verified | Reset's own verification step | At job boundary |

The event listener runs a single goroutine that subscribes to the backend's event stream filtered by the sandbox's container label, marking matching executors `dead` in their pools. The listener reconnects with exponential backoff on transient errors, and shutdown waits for the listener to exit cleanly.

The acquire hot path scans `dead` first (a single memory read), then performs a single filesystem read for the heartbeat. Both checks short-circuit the slot from selection and queue it for background destruction. The destruction runs on the pool's wait group, ordered with respect to pool removal so that a recreated pool with the same key cannot race against an orphan teardown.

## Per-job resource overrides

The pool's configuration sets default memory limit, CPU limit, and timeout. The per-question execution config can override memory, CPU, timeout, filename, and command. At `Exec` time, if the override differs from the container's currently applied limits, the executor updates the container's resources via the backend API before running the command and reverts to pool defaults after the command completes (using a non-cancellable context for the revert, so a cancelled parent context does not skip cleanup). If the revert fails, the executor's bookkeeping holds the value at the override, so the next `Exec` re-applies pool defaults on its way to whatever the next override is.

Backend choice is deliberately **not** an override. It is fixed at extension-instance level; operators that need heterogeneous backends run multiple sandbox extension instances.

## Hot configuration updates

When an instructor edits a pool's image or resource limits, the running pool must transition without dropping queued jobs.

1. **Pull the new image before tearing down executors** ("make before break"). The pull happens between two `Lock` / `Unlock` cycles so the pool's mutex is never held across the pull.
2. **Idle executors** for the new image are destroyed immediately and recreated on the next acquire.
3. **Busy executors** are marked `pendingRecreate`. On their next release they are routed to the destroy-and-replace path, and if a waiter is queued, a replacement container is provisioned for that waiter.
4. **Tag policy.** Image references tagged `:latest` (or untagged) are always pulled even if a local copy exists — local presence does not prove the remote has not moved. Pinned tags or digests are pulled only when missing locally.

## Job tracking

Each submission is allocated a ULID and stored in shared cache with a 10-minute TTL under two keys: `sandbox:job:<jobID>` (the full job record) and `sandbox:user:<u>:doc:<d>:q:<q>` (an index mapping the active `(user, document, question)` triple to its job ID).

The "at most one active job per `(user, document, question)`" invariant is enforced atomically by the tracker: the index key is written with a SET-NX-with-TTL primitive at job-creation time, so two simultaneous submissions race for the same key and exactly one wins. The loser receives an error carrying the existing job's ID, which the API layer surfaces as a 409 response so the client can attach to the in-flight job rather than create a duplicate. The full job record is written only after the index claim succeeds; a failure to write the record rolls back the index claim so the user can immediately retry.

Cleanup: the tracker's `GetActiveJobForUserQuestion` self-heals by removing index entries whose pointed-to job has reached a terminal state, so a stale claim from a crashed job cannot block a retry beyond the next lookup. The 10-minute TTL bounds the worst-case stale-entry lifetime when no lookup occurs.

## Linter (peer system, distinct shape)

The linter is a sibling concern: instructor-configured static analysis that runs on a code snippet and returns structured issues. Linting is short, idempotent, and stateless — no per-job filesystem isolation is needed. The current shape is **one** singleton container with an in-process semaphore bounding concurrency; commands `tee` the snippet into a UUID-prefixed file in `/tmp`, run the linter on that file, and remove it. The linter is therefore not part of the warm pool but is part of the sandbox extension's surface; it is included here for completeness, not for symmetry.

When future backends land, the linter folds onto the same backend abstraction as a one-slot pool, dropping the second backend client. The semaphore semantics translate cleanly to the existing pool waiter mechanism.

## Execution backend

Today the sandbox is built directly against the Docker SDK. This section describes the Docker backend as implemented and the **target abstraction shape** for the multi-backend follow-up. The `Backend` interface described below does not exist in the codebase today; it is the contract this RFD commits to as the seam along which backend additions will land.

### Target interface shape (not yet implemented)

The interface, expressed informally:

- `EnsurePool(key, config)` — make whatever supporting resources the backend needs (image pull, namespace creation, etc.).
- `AcquireSlot(key)` / `ReleaseSlot(key, slot, dirty)` — map onto the pool state machine.
- `Exec(slot, command)` — run a command in the slot, return stdout / stderr / exit code / timing / memory / OOM signal.
- `Reset(slot)` — bring a slot back to the freshly-acquired state. Backends may implement via in-place cleanup, slot replacement, or any other mechanism; the contract is the post-condition, not the steps.
- `Health(slot)` — cheap liveness check, allowed to be a filesystem read or a memory read.
- `PeakMemoryMB(slot)`, `OOMKilledSinceLast(slot)` — per-`Exec` resource accounting.
- `WriteFile(slot, path, content)` — write the student code into the slot.
- `UpdatePool(key, newConfig)` / `RemovePool(key)` — hot-update and teardown.
- `Type()` / `Stats(key)` — observability.

The pool manager, idle-eviction watcher, hot-config updater, event listener, and acquire/release waiters are designed to be backend-agnostic once the interface is extracted; the per-backend code will then be the implementation of these methods plus the data structures the methods need. Extracting the interface is a refactor of the existing pool manager, not a rewrite — the entry/exit points listed above already correspond one-to-one to the methods the pool manager calls today.

### Docker backend (current)

The Docker backend is the only implementation today. It uses the standard Docker SDK against a daemon reachable through `DOCKER_HOST` (host socket by default).

- **Pool lifecycle**: image pull through the registry service (which understands ZINC's registry addressing); container create with the security configuration described above; container start with a post-start inspect that surfaces an immediate-death diagnostic if a security policy refused the helper at start time.
- **Slot acquisition**: idle scan in process memory, plus a fast event-listener-fed `dead` bit, plus a host filesystem heartbeat read.
- **Exec**: `ContainerExecCreate` + `ContainerExecAttach` with the ghost wrapping rule. Output is read from the host bind-mount for the ghost path and from the demultiplexed attach stream for the root path.
- **Reset**: the verified post-condition sequence described above, all via `ContainerExec*`.
- **PeakMemory / OOM**: the host reads the container's cgroup `memory.peak` (cgroup v2) or `memory.max_usage_in_bytes` (cgroup v1) via the path resolved at start from the container's init PID. The OOM counter is sampled from the same cgroup directory. Both rely on the host and container sharing a kernel.
- **Event listener**: subscribes to the daemon's events stream filtered by the sandbox label and the `die` / `oom` actions, with exponential-backoff reconnect.
- **Output sink**: per-executor host scratch directory.

Two coupling points to the Docker daemon's host machine are explicit:

- The output sink is a host bind-mount.
- Peak memory and OOM tracking read host cgroup files resolved from the container's init PID.

These are the load-bearing assumptions that the backend abstraction allows other backends to break — without rewriting the pool manager.

### Other backends

Kubernetes and Nomad backends are out of scope for this RFD; each will be covered by its own follow-up RFD. Those RFDs will need to address — at minimum — how the warm pool maps onto the backend's primitives, how output capture works when the host and the workload don't share a filesystem, how peak memory and OOM are reported when the host can't read the workload's cgroup directly, how dead workloads are detected from Pod / alloc state transitions rather than Docker events, and how seccomp / AppArmor / RBAC are configured against each backend's API surface. Each follow-up RFD also re-runs the warm-vs-cold benchmark from [Empirical justification of the warm pool](#empirical-justification-of-the-warm-pool-docker-backend) against its primitives: the Docker-host numbers in this RFD are not transportable to distributed backends, and the architectural conclusion — that warm pooling wins per-job — must be confirmed by measurement, not assumption.

The interface and the pool manager's state machine are deliberately shaped so that adding a backend is purely additive: the Docker backend implementation, the pool's lifecycle code, and the security configuration model do not need to be reshaped.

## API and data surface

The sandbox extension exposes an HTTP surface mounted under `/v1`, structured around four contract-level operations. All routes require an authenticated session (`RequireAuthenticated`); paths use the `examination` source's `document`/`question` keying.

| Operation | Method + path | Purpose |
|---|---|---|
| Submit | `POST /sandbox/executor/documents/:document_id/questions/:question_id/execute` | Submit a code snippet for execution; returns a job ID. Rejects with 409 if an active job already exists for the same `(user, document, question)`. |
| Observe | `GET /sandbox/jobs/:job_id/status` + `GET /sandbox/jobs/:job_id/stream` | Poll or stream a job's state through to a terminal result. Stream is server-sent events. |
| Active-job lookup | `GET /sandbox/executor/documents/:document_id/questions/:question_id/active-job` | Returns the in-flight job for `(user, document, question)` so a reloading client can re-attach. |
| Lint | `POST /sandbox/lint/documents/:document_id` | Request a lint pass on a snippet against the document's lint configuration. |

Beyond these contract operations, the extension also exposes:

- **Attempt-history CRUD** under `/sandbox/executor/documents/:document_id/questions/:question_id/attempts*` (count, listing, individual item retrieval, coordinator-gated deletion) backing the student "previous attempts" UI.
- **Pool / per-question configuration CRUD** under `/sandbox/executor/activities/:activity_id/config` and `/sandbox/executor/documents/:document_id/questions/:question_id/config`, gated by `can_edit` on the activity.
- **Observability**: `GET /sandbox/executor/documents/:document_id/executors/stats` for pool snapshots.

These are conventional resource CRUD and not load-bearing to the design; they are listed for completeness and not specified further here.

Authorization is enforced through the existing OpenFGA-backed middleware: `student` on the activity (or higher) for execute / poll / stream / lint; coordinator-level for configuration CRUD.

The pool and question-execution-config tables live alongside the other sandbox tables in the shared database. The pool config is updated by an event (`environment.template_changed`) flowing through the broker, which hot-swaps the pool config without requiring the extension to redeploy.

## Known limitations and accepted trade-offs

- **Warm-pool resource reservation.** A pool slot reserves CPU and memory while idle. The trade-off is bought by latency. Operators are expected to size `executor_limit` for steady-state demand, not peak, and the idle-eviction watcher reclaims slots after a long idle threshold.
- **Per-`Exec` cgroup write requires root or pre-relaxed permissions.** The peak-memory reset (cgroup v2 `memory.peak` write) requires write access to the cgroup file. Where the sandbox extension does not run as root, the operator must arrange for the cgroup files to be writable by the extension's UID. The implementation falls back to `sudo -n` if the direct write fails.
- **Pool slot count is fixed at `executor_limit` for the pool's lifetime.** Pools warm up to `executor_limit` slots on first acquire and keep all of those slots alive until the pool itself is idle-evicted. For CPU pools this is correct: idle slots cost a few percent CPU and a few hundred MiB of RAM, and the alternative would lose the warm-pool latency property for sporadic users.
- **AppArmor coverage is host-dependent.** Hosts without AppArmor (most CI environments, some minimal distributions) run with seccomp and the other layers but not AppArmor. The defence-in-depth model treats this as a degradation, not a failure.
- **Output cap kills the whole container, not just the command.** A student that exceeds the per-`Exec` output cap triggers a container kill and a slot replacement; the next acquire pays the cold-start cost for that one slot. This is deliberate — closing the exec stream is insufficient to revoke the held file descriptor, and the alternative (no cap) is "fill the host disk."

## Abandoned ideas

A few alternatives surfaced during design that we chose not to pursue. Recording them so the same paths aren't re-walked.

### Per-job container (no warm pool)

Conceptually simplest: spawn a fresh container per submission, run the code, destroy it. The benchmark in *Empirical justification of the warm pool* settles this — the daemon-side lifecycle (seccomp install, AppArmor profile apply, cgroup setup, tmpfs mounts, network-namespace setup, image inspect) dominates a trivial `Exec` by roughly 6×, and the overhead is not amortisable across a single interactive submission. The warm pool's complexity (Reset, dirty release, state machine) buys back two orders of magnitude of headroom on per-replica throughput.

### Count-based recycling instead of verified Reset

An earlier draft proposed retiring containers after a fixed number of jobs (e.g. "recycle every 100 uses"). Discarded because the failure mode is the wrong shape: state leaks between students happen at use *N+1*, not at use 100, and a counter cannot tell whether a particular job left state behind. The verified-Reset model measures the actual post-condition (kill verified, wipe verified) and only reuses a container when both pass. Counters are a poor proxy for "is this container clean."

### Stronger isolation primitives (Kata, gVisor) as the load-bearing layer

Replacing the seccomp + AppArmor + capability + namespace stack with a VM-style isolation primitive (Kata Containers, gVisor) was considered. Discarded as the load-bearing layer for two reasons. First, the warm-pool latency budget is sub-100ms; Kata's cold-cycle probably seconds even with template VMs, which closes the latency gap that warm-pooling exists to open. Second, the failure mode of those substrates (kernel-API surface differs subtly from a real kernel, syscall coverage incomplete) means that real student code — particularly anything that uses recent libc features — risks running differently than on the target environment. They remain candidates as an *additive* layer (a Kata-on-Docker backend), but not as a replacement.

### App-level cross-replica pool ownership

Considered routing requests for the same `(source, sourceID)` to a "leader" replica that owns the pool, with replicas coordinating ownership through Valkey leases. Discarded because the same outcome is achieved by the upstream load balancer's consistent-hash routing on `(source, sourceID)`, without any app-level leadership protocol. Pool ownership stays implicit: whichever replica receives the request creates or reuses the pool locally; affinity is the LB's responsibility.

### Single global `dirty` slot state

Considered adding `dirty` as a fourth primary slot state, parallel to `creating`/`idle`/`busy`. Discarded in favour of the present design — `dirty` is a transient signal on the release path, not a state a slot ever sits in for any duration. Modelling it as a per-release flag that mutates `pendingRecreate` is closer to the actual lifecycle: a slot is either being held by a job (`busy`), available (`idle`), being built (`creating`), or marked for destruction (`pendingRecreate`). The "Reset failed" condition collapses into the existing destruction path rather than introducing a fourth state.

## Glossary

| Term | Definition |
|---|---|
| **Pool** | A set of warm executor slots keyed by `(source, sourceID)`, sized by instructor configuration. |
| **Executor** | One long-lived container in a pool. Has a stable identity and a primary state (`creating`, `idle`, `busy`) plus two orthogonal modifier flags (`dead`, `pendingRecreate`). |
| **Slot** | An abstract reservation of capacity in a pool. Backed by an executor on the Docker backend; backed by a Pod / alloc on future backends. |
| **Backend** | The target abstraction behind the pool's lifecycle and exec primitives. Today: Docker (directly, no interface boundary yet). Future: a `Backend` interface plus Kubernetes / Nomad implementations. |
| **ghost** | A small static binary baked into the sandbox image. Acts as the container's PID 1 (heartbeat mode) and as a per-`Exec` wrapper (run mode), providing in-container isolation that complements the host-side container configuration. |
| **Reset** | The verified post-condition that returns an executor to the freshly-acquired state between two students' jobs. |
| **Cold-start cycle** | The full lifecycle a per-job-spawn implementation would pay: container create + start + exec + stop + remove. |
| **Warm execution** | The execution path the warm pool uses: a single `Exec` against an already-running, already-isolated container. |
| **Dirty release** | A release where the caller signals that the executor *might* still hold previous-job state. Routed to destroy-and-replace, never reused. Modelled as a per-release argument, not a stored slot state. |
| **Output cap** | The per-`Exec` limit on the on-disk size of stdout + stderr while the student's code is running, enforced by a watchdog that kills the container on overshoot. |

## References

- [RFD 0009 — ZINC Extension System](../0009/README.md)
- [RFD 0010 — Temporal Workflow Orchestration Integration](../0010/README.md)
