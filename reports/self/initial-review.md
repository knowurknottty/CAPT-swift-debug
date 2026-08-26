# Self Review — CAPT / Inversion Labs CAPT macOS Swift

## Overall status
- **Standard CAPT:** builds and unit-tests cleanly, including strict-concurrency, TSan, and ASan probes, but is **not release-clean** because confirmed liveness/IPC/subprocess and response-contract defects remain.
- **Inversion Labs CAPT:** same core build health, plus Labs-specific UI/model additions, but the installed app was built from **six uncommitted Swift/test files**, so the installed binary is not reproducible from its recorded commit. The separate `dual-swift-labs-r1` integration worktree is also unresolved/conflicted and is not merge-ready.
- **No evidence-backed P0 found.** Several external-review P0 claims were rejected after source/test adjudication.

## Confirmed findings

### SELF-001 — P1 — Runtime model output contract mismatch
**Evidence:** `Sources/CAPTCoreDesktop/CAPTChatCoordinator.swift:174-196` extracts `observations[].content`, while the current CAPT provider/Hermes driver contract returns `observations[].summary`. Current Python runtime receipt construction preserves those observations under `result.observations`.

**Reproduction:** an added audit-only regression test supplied the current runtime shape `result.observations[0].summary = "CAPT says hello"`; the Swift coordinator returned the pretty-printed entire JSON receipt instead of `CAPT says hello` and the test failed exactly on that assertion.

**Impact:** successful model runs can render raw receipt JSON instead of the assistant answer in the macOS chat surface.

**Fix:** teach `extractAssistantText` to prefer `summary` as well as `content`, both top-level and inside `result.observations`; preserve backward compatibility with existing `content` tests. Add contract tests using the real runtime receipt shape.

### SELF-002 — P1 — `CAPTOperatorCLI.run` can deadlock on child pipe backpressure
**Evidence:** `CAPTOperatorCLI.swift:202-217` connects both stdout/stderr to `Pipe`, calls `process.waitUntilExit()` first, and drains both pipes only after child exit.

**Reproduction:** an audit-only fake `capt-ui` wrote enough output to fill the pipe. The XCTest remained blocked for more than **7 hours** with the child still alive and blocked writing output. The probe was manually terminated after confirmation.

**Impact:** provider/model/operator commands can hang the macOS app indefinitely when a CLI command emits sufficiently large output or diagnostics.

**Fix:** drain stdout/stderr concurrently while the process runs, or use asynchronous readability handlers / a bounded communicate helper; add a process timeout and output-size cap.

### SELF-003 — P1 — Unix-socket transactions have no receive/send deadline
**Evidence:** `CAPTRuntimeClient.swift` performs blocking `Darwin.send`/`Darwin.recv` loops under a single `NSLock`; no `SO_RCVTIMEO`, `SO_SNDTIMEO`, poll/select deadline, or cancellation path exists.

**Reproduction:** audit server completed authentication and identity, accepted a capabilities request, then withheld the response frame. `CAPTRuntimeClient` did not return within the external 4-second test deadline (`TIMEOUT_CONFIRMED`).

**Impact:** a wedged or partially responsive RuntimeService can strand the background actor and leave operator actions permanently waiting until app/process restart.

**Fix:** use explicit socket read/write deadlines and a typed timeout error; close/reset the socket on timeout. Add partial-header, partial-body, and silent-server tests.

### SELF-004 — P1 — Installed Labs build is not source-reproducible
**Evidence:** the Labs installed-source snapshot was built immediately after six modified files changed (`CAPTChatCoordinator.swift`, `CAPTRuntimeModels.swift`, `ChatView.swift`, `InspectorView.swift`, and two test files). The captured working-tree patch adds selected `skillNames` propagation/display and backward-compatible decoding. `labs-source-status.txt` records all six as modified.

**Impact:** the installed Labs binary cannot be reproduced from branch HEAD alone; rollback, bisect, audit, and release provenance are weakened even though the delta itself is reasonable and tested.

**Fix:** commit the exact installed delta (or rebuild from a clean committed revision), record the source SHA in app/build metadata, and require clean-tree release builds.

### SELF-005 — P1 integration blocker — Labs convergence worktree is unresolved
**Evidence:** `dual-swift-labs-r1` contains unresolved `UU`/`AA` conflicts across core runtime and Swift integration files, including `checkpoint.py`, `composition.py`, `store.py`, `provider.py`, `model_approval_binding.py`, `prompt_approval.py`, `CAPTOperatorCLI.swift`, `CAPTRuntimeBootstrapper.swift`, `CAPTBackgroundRuntime.swift`, `CAPTOperatorStore.swift`, build/install scripts, and others.

**Impact:** that integration lane is not mergeable or trustworthy as a release source until conflicts are explicitly reconciled and reverified.

**Fix:** reconcile against current authoritative main/Labs lineage, then rerun full Python + Swift + cross-surface gates before merge.

## Material P2 risks / release gaps

### SELF-006 — P2 — Runtime bootstrap has no wall-clock timeout
`CAPTRuntimeBootstrapper.swift:64-83` calls `process.waitUntilExit()` with output suppressed and no deadline. A hung `capt start` makes reconnect/bootstrap wait forever. Add a deadline, terminate/kill escalation, and explicit timeout error.

### SELF-007 — P2 — Session store claims unchecked Sendability without serialization
`CAPTEncryptedSessionStore` is `@unchecked Sendable` and exposes synchronous `load`/`save` without internal locking. Startup deliberately calls `load()` from `Task.detached`, while main-actor flows may call `save()` before restore completes. Atomic file replacement protects against torn files, but overlapping load/save can still produce stale-merge/lost-update behavior. Serialize access with an actor/lock and test concurrent restore + new-chat/save.

### SELF-008 — P2 — Installed apps are development builds, not distribution/notarization proof
Both installed bundles pass `codesign --verify --deep --strict`, but are signed with an **Apple Development** identity, CodeDirectory flags are `0x0` (no hardened-runtime flag), and `spctl -a -vv --type execute` rejects both. Build scripts sign without `--options runtime`, notarization, or stapling. This is acceptable for local development but not a release/distribution claim.

### SELF-009 — P2 verification gap — live/cross-surface tests are skipped by default
Frozen snapshot tests pass (`64` tests each; Standard `7` skipped, Labs `4` skipped). `CAPTLiveRuntimeTests` requires `CAPT_LIVE_TEST=1`; Standard cross-surface acceptance also requires `CAPT_CROSS_SURFACE_TEST=1` and stage selection. Unit/sanitizer green status therefore does not establish live RuntimeService/MCP/macOS acceptance.

## Build / sanitizer evidence
- Standard: normal `swift test` + release build PASS; strict-concurrency build PASS; TSan rerun PASS; ASan PASS.
- Labs: normal `swift test` + release build PASS; strict-concurrency build PASS; TSan PASS; ASan PASS.
- The original Standard TSan failure was an index-store temporary-file collision caused by concurrent build activity; serialized rerun passed and is the authoritative sanitizer result.

## Rejected / downgraded external claims
- **“`Task {}` inside `@MainActor CAPTOperatorStore` violates MainActor” — rejected.** Unstructured tasks created in actor-isolated context inherit actor isolation; strict-concurrency build also passes.
- **“`submitPrompt` ignores `beginPrompt` failure” — rejected.** Current source uses `guard let sessionID = mutateWorkspace({ beginPrompt(...) }) else { return }`.
- **“Changing provider/model/target does not invalidate approval” — rejected.** `CAPTNativeChatWorkspace.updateConfiguration` calls `supersedeApproval` when configuration changes with pending/requesting approval.
- **“Missing `kSecAttrAccessible` in Keychain read causes item-not-found/key regeneration” — rejected.** Accessibility class is an item attribute, not required as a lookup selector; the claimed failure mechanism is unsupported.
- **“AfterFirstUnlockThisDeviceOnly causes key regeneration after reboot before unlock” — rejected as stated.** Locked/inaccessible Keychain errors are not equivalent to `errSecItemNotFound`; code only creates a new key on item-not-found.
- **“approveAndRun can bypass expired approval authority” — downgraded.** The native workspace validates actionability before calling it, and RuntimeService remains authoritative/fail-closed. A direct coordinator caller may cause a rejected network attempt, but not unauthorized execution.

## Current verdict
**FIX before declaring either macOS edition production-ready.** The strongest shared blockers are SELF-001/002/003. Labs additionally needs SELF-004 and integration reconciliation before release provenance can be considered clean.
