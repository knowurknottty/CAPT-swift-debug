# Hermes ExecutionDriver Analysis Artifact

DriverRunId: dr-model-fd253b32e7335bfc15bb84e2
Target: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs
ExternalPid: 35907
ExitCode: 0
ElapsedSeconds: 160.79

## Untrusted runtime output

# CAPT Swift/macOS Adversarial Code Review
## Target: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs
## Scope: READ-ONLY inspection of all .swift source files

### Critical Findings (P0/P1)

**1. CAPTRuntimeClient: No timeout on socket recv [P0, high confidence]**
- `receiveUnlocked()` at `CAPTRuntimeClient.swift:152` calls `Darwin.recv()` with no timeout
- `sendUnlocked()` at line 128 has same issue — `while offset < frame.count` `Darwin.send()` loop blocks indefinitely
- **Mechanism**: If RuntimeService stops responding or network interference, `CAPTRuntimeClient` threads hang forever. No cancellation mechanism exists.
- **Impact**: Complete client freeze; all subsequent operations blocked until process termination.
- **Fix**: Add `DispatchWallTimer`/`withTimeout` or `NWConnection` with deadline; propagate cancellation via `Task.interruption()`.

**2. CAPTBackgroundRuntime: Bootstrapper start() blocks indefinitely [P1, high confidence]**
- `start()` at `CAPTRuntimeBootstrapper.swift:95` calls `process.waitUntilExit()` with no timeout
- If `capt start --state-dir` hangs (e.g., missing dependencies, locked state), the entire bootstrapping sequence freezes
- **Mechanism**: `Process.run()` + `waitUntilExit()` with no wall-clock deadline; `process.terminationStatus` check only after exit.
- **Impact**: App cannot recover from runtime bootstrap failure; user must kill/force-quit.
- **Fix**: Add `process.waitUntilExit(timeout:)` with `DispatchTime.now() + 30`; distinguish exit-timed-out vs normal exit.

**3. CAPTOperatorStore: Task nesting races in submitPrompt/approvePending [P1, medium confidence]**
- `submitPrompt()` at `CAPTOperatorStore.swift:218` mutates workspace then starts Task that reads `pendingApproval` captured by closure
- `approvePending()` at line 338 similarly captures `localPending` then mutates workspace inside Task
- **Mechanism**: `mutateWorkspace` copies chatWorkspace, but Task closure may capture stale reference; `activeSessionID == sessionID` guard checks after Task dispatch, not atomic
- **Impact**: Approval flow can proceed with stale provider/model/targetRoot; expired approvals dispatched; configuration drift.
- **Fix**: Capture `self` weakly in Tasks; re-read `pendingApproval` from `chatWorkspace` at await point; use `async let` for parallel reads.

**4. CAPTNativeChatWorkspace: Flow dictionary stale entries on session merge [P2, medium confidence]**
- `mergeRestoredSessions()` at `CAPTNativeChatWorkspace.swift:93` appends flows: `flows[restoredSession.id] = CAPTChatFlow(pending: restoredSession.pendingApproval, now: now)`
- No removal of prior flow entries for same ID; `flow(for:)` at line 136 returns `flows[id] ?? CAPTChatFlow(pending: session(id)?.pendingApproval)` — may return old flow if ID reused
- **Mechanism**: Session IDs are UUIDs (collision-resistant), but restored sessions from encrypted store may have stale `pendingApproval` state
- **Impact**: Wrong approval flow displayed; expired approvals treated as valid; configuration mismatches.
- **Fix**: Flush `flows[id]` before insertion: `flows[id] = nil; flows[id] = ...`; or add version stamp to `CAPTChatFlow`.

**5. CAPTRuntimeClient: No re-authentication on reconnect [P2, medium confidence]**
- `connect()` at `CAPTRuntimeClient.swift:82` calls `disconnect()` then re-establishes socket + sends token
- If socket connect succeeds but server rejects auth mid-session, `disconnect()` is called but no re-auth on next `connect()`
- **Mechanism**: `disconnect()` sets `operatorID = nil, sessionID = nil` but next `connect()` re-reads token file and re-sends; however if `connect()` called from `CAPTBackgroundRuntime.connect()` at line 33, bootstrapper starts then retries — no auth state reset between attempts.
- **Impact**: Stale credentials accepted; silent auth failures; operations silently fail after apparent connect success.
- **Fix**: Reset `operatorID`, `sessionID` before re-auth; explicit `disconnect()` → `connect()` handshake reset.

**6. CAPTEncryptedSessionStore: @unchecked Sendable with potential data race [P2, medium confidence]**
- `CAPTEncryptedSessionStore` at `CAPTNativeSessionStore.swift:58` marked `@unchecked Sendable`
- `keyProvider.keyData()` calls `SecItemCopyMatching/SecItemAdd` — not Sendable-safe across actor boundaries
- `save()`/`load()` read/write `fileURL` — concurrent access from multiple `Task { }` contexts (e.g., `restoreSessionsAsync` + `saveSessions`) can corrupt encrypted file
- **Mechanism**: `@unchecked Sendable` tells compiler "this is thread-safe" but AES-GCM sealed box operations + file I/O are not inherently atomic; `SecItemAdd` is not reentrant-safe.
- **Impact**: Corrupted encrypted sessions; data loss; decryption failures.
- **Fix**: Remove `@unchecked Sendable`; add `actor CAPTEncryptedSessionStore` for serialized access; or add `DispatchSemaphore` around file operations.

**7. CAPTOperatorCLI: No working directory isolation [P3, low confidence]**
- `run()` at `CAPTOperatorCLI.swift:136` spawns `Process()` with `executableURL` and `arguments` only
- No `currentDirectoryURL` set; subprocess inherits VM working directory
- `targetRoot` paths passed as CLI arguments could contain `..` components resolved relative to CWD
- **Mechanism**: `capt-ui` binary invoked from unpredictable CWD; path traversal in argument parsing
- **Impact**: Unexpected file access; command injection via crafted paths in `labInputJSON` forwarded as CLI args.
- **Fix**: Set `process.currentDirectoryURL` to a sandboxed temp dir; validate/escape all path arguments.

### Design Risks (P2/P3)

**8. CAPTOperatorStore: providerCredentialStatus persists across disconnect/reconnect [P3, low confidence]**
- `providerCredentialStatus[providerID]` set during `testProvider`/`configureProviderAPIKey` never cleared on `disconnect()`
- After runtime restart, stale "Authenticated ✓" status may be displayed for disconnected state
- **Mechanism**: Published property written to `lastError` only; no `onAppear`/`onDisappear` lifecycle cleanup
- **Impact**: UI lies about authentication state; user surprised when "authenticated" provider fails.

**9. CAPTRuntimeClient: No FD_CLOEXEC on socket [P3, low confidence]**
- `openUnixSocket()` at `CAPTRuntimeClient.swift:216` creates socket with `Darwin.socket(AF_UNIX, SOCK_STREAM, 0)` — no `FD_CLOEXEC` flag
- If `capt start` subprocess is launched from same process, socket FD could be inherited
- **Mechanism**: Default socket behavior inherits FDs across `exec;` relevant if `CAPTBackgroundRuntime.start()` forks.
- **Impact**: Resource leak; potential FD exhaustion in long-running deployments.
- **Fix**: `fcntl(fd, F_SETFD, FD_CLOEXEC)` after socket creation.

**10. CAPTNativeChatWorkspace: no invalidation on targetRoot/config change [P3, low confidence]**
- `updateConfiguration()` at `CAPTOperatorStore.swift:278` calls `persistConfiguration` then sets `provider = newProvider; model = newModel; targetRoot = newTargetRoot`
- No automatic `supersedeApproval` if `pendingApproval` exists with different config
- **Mechanism**: User changes provider/model in UI; old `pendingApproval` bound to previous config remains "valid" per `validity(at:)` check only
- **Impact**: Approval dispatched with wrong provider/model; silent execution failure.
- **Fix**: `persistConfiguration` already calls `supersedeApproval` when `pendingApproval != nil || phase == .requestingApproval` — verify this path is exercised.

### Verified Defects (Evidence-Backed)

| ID | Severity | Confidence | File:Line | Mechanism | Impact | Fix |
|---|---|---|---|---|---|---|
| D1 | P0 | high | `CAPTRuntimeClient.swift:152` | `Darwin.recv()` no timeout | Client hang | Add deadline/timeout |
| D2 | P1 | high | `CAPTRuntimeBootstrapper.swift:95` | `process.waitUntilExit()` no timeout | Bootstrapper freeze | Add wall-clock timeout |
| D3 | P1 | medium | `CAPTOperatorStore.swift:218,338` | Task closure stale state capture | Approval flow drift | Re-read state at await point |
| D4 | P2 | medium | `CAPTNativeChatWorkspace.swift:93` | Stale flow entries on merge | Wrong approval flow | Flush flows[id] before insert |
| D5 | P2 | medium | `CAPTRunt...
