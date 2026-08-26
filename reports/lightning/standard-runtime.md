# Hermes ExecutionDriver Analysis Artifact

DriverRunId: dr-model-be66c0e98aca18913abeb7ed
Target: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/standard
ExternalPid: 34195
ExitCode: 0
ElapsedSeconds: 195.96

## Untrusted runtime output

## CAPT Swift/macOS Audit — Standard Snapshot (2026-08-25)

### [RACE-01] CAPTRuntimeClient: No socket read timeout — potential hang/DoS
- **Severity:** P1  | **Confidence:** high  | **File:** CAPTRuntimeClient.swift:138-149
- **Evidence:** `readExact(_:)` uses `Darwin.recv` with no timeout parameter; blocks indefinitely if server stops responding or network stalls
- **Mechanism:** Client call `try client.query(...)` or `try client.command(...)` never returns; watchdog/process manager must kill the task
- **Impact:** Liveness failure — operator cannot cancel or recover without process restart
- **Fix:** Add `setsockopt` SO_RCVTIMEO before `recv` calls; wrap `readExact` in timeout guard
- **Test:** Verify `testOversizedFrameLengthFailsClosed` style: assert `readExact` throws after configurable timeout when socket is live but unresponsive

### [RACE-02] CAPTChatCoordinator: `approveAndRun` dispatches without checking approval actionability
- **Severity:** P0  | **Confidence:** high  | **File:** CAPTChatCoordinator.swift:180-192
- **Evidence:** `approveAndRun:` calls `client.command(op: "run_approved_hermes_inspection", ...)` without calling `pending.isActionable(at:)` — proceeds even if approval expired since `requestApproval`
- **Mechanism:** Timer-driven expiry on server side + client has no fresh validity check before network dispatch
- **Impact:** Dispatch with stale/expired approval — system may reject or accept undefined state; violates approval binding contract
- **Fix:** Add `guard pending.isActionable(at: Date()) else { throw CAPTRuntimeClientError.malformedResponse("approval expired or consumed") }` before the `run_approved_hermes_inspection` command
- **Test:** Simulate expiry: create approval, wait past `expiresAt`, call `approveAndRun`; assert `CAPTRuntimeClientError.malformedResponse` thrown

### [RACE-03] CAPTChatFlow: `beginExecution` transitions to `.executing` without validity check
- **Severity:** P1  | **Confidence:** high  | **File:** CAPTChatFlow.swift:118-127
- **Evidence:** `beginExecution(_:)` sets `phase = .executing` regardless of `pending.validity(at: now)` — no `.valid` guard
- **Mechanism:** Flow state advances before runtime confirms approval validity; subsequent `executionFailed` may classify incorrectly
- **Impact:** State mismatch — UI shows "executing" but runtime may reject; incorrect disposition cascades
- **Fix:** Add `guard case .valid = pending.validity(at: now) else { return }` at `beginExecution` entry, or shift validity check to caller
- **Test:** Create flow with expired approval; call `beginExecution`; assert phase stays `.awaitingApproval` or transitions to `.recoverableFailure`, not `.executing`

### [RACE-04] CAPTNativeSessionStore: Keychain `AfterFirstUnlockThisDeviceOnly` breaks after reboot
- **Severity:** P2  | **Confidence:** high  | **File:** CAPTNativeSessionStore.swift:53-58
- **Evidence:** `keyData()` uses `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`; if device reboots without unlocking, `SecItemCopyMatching` returns `errSecItemNotFound`, triggering `createKey()` which overwrites the key
- **Mechanism:** Key rotation on every reboot decrypts existing sessions as garbage; all `classic_native_sessions.enc` content lost
- **Impact:** Data loss — all cached chat sessions and approvals irrecoverable after reboot
- **Fix:** Use `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` (available on iOS/macOS) or add fallback: try `AfterFirstUnlock`, on `ItemNotFound` try `WhenUnlocked`, then generate new key only if both fail
- **Test:** Bootstrapped macOS environment: set keychain item with `AfterFirstUnlockThisDeviceOnly`, reboot simulator without unlock, attempt `load()` — verify either decryption succeeds or fails gracefully without overwriting

### [RACE-05] CAPTRuntimeClient: Socket FD leaked on `connect()` error paths
- **Severity:** P1  | **Confidence:** medium  | **File:** CAPTRuntimeClient.swift:86-107
- **Evidence:** `openUnixSocket` succeeds at line 96, then `send` fails at line 102 — `disconnect()` is never called; `socketFD` remains open
- **Mechanism:** `connect()` returns after `send` failure; `deinit` calls `disconnect()` but SwiftARC may not fire if client retained elsewhere
- **Impact:** File descriptor leak — cumulative across sessions; eventual `EMFILE`/`too many open files` if many connect/disconnect cycles
- **Fix:** Wrap `connect()` body in `{ defer { disconnect() } }` or add `defer { if socketFD >= 0 { Darwin.close(socketFD); socketFD = -1 } }` after `openUnixSocket`
- **Test:** Allocate client, call `connect()` with bad socket path, verify `socketFD == -1` afterward; repeat 100× and assert `task ports` count stable

### [RACE-06] CAPTProviderCLI: `requiresPrewarm` too narrow — misses local-ollama prewarm
- **Severity:** P3  | **Confidence:** medium  | **File:** CAPTOperatorCLI.swift:218-224
- **Evidence:** `requiresPrewarm` requires `kind.lowercased() == "local" && transport == "openai_compatible"` — local `ollama` providers (transport=`ollama`) never trigger prewarm even though MLX-backed local models may need it
- **Mechanism:** PrewarmArguments only invoked when this returns `true`; warm skipped; model load may be delayed or fail first time
- **Impact:** Suboptimal first-use latency for local-ollama setups; no warmup invoked even when beneficial
- **Fix:** Broaden condition to include `transport == "ollama"` or add separate `requiresPrewarmForTransport` logic
- **Test:** Decode local ollama provider; assert `requiresPrewarm(provider, modelID: "m")` returns `true`; currently returns `false`

### [RACE-07] CAPTChatFlow: `executionFailed` classification gated on message-content keywords
- **Severity:** P2  | **Confidence:** medium  | **File:** CAPTChatFlow.swift:81-88 + CAPTApprovalRecoveryPolicy.swift:12-23
- **Evidence:** `CAPTApprovalRecoveryPolicy.classify` inspects error message text for substrings (`"prompt_approval_expired"`, `"approval_consumed"`, `"approval_denied"`). An error containing these as substrings but meaning differently triggers wrong disposition
- **Mechanism:** False-positive keyword match → `.expired`/`. consumed`/`.`denied` classification; `beginExecution`/local cursor state becomes inconsistent
- **Impact:** Approval cursor incorrectly retired; user must re-submit prompt unnecessarily, or worse, proceed with consumed approval
- **Fix:** Decouple classification from raw message text; have RuntimeService return a structured `approvalStatus` field in `submit_approval_decision` response; classify on that instead
- **Test:** Craft error message `"some other error: prompt_approval_expired context"`; assert classify returns `.retryable` not `.expired` (or fix to exact match)

### [RACE-08] CAPTNativeSessionStore: No ciphertext format versioning
- **Severity:** P3  | **Confidence:** medium  | **File:** CAPTNativeSessionStore.swift:112-132
- **Evidence:** `decodeSessions(at:)` attempts AES-GCM open; on failure throws `CAPTSessionStoreError.malformedCiphertext`. No version byte/field in sealed box; format changes indistinguishable from corruption
- **Mechanism:** If encryption format evolves, all existing sessions appear corrupted; no migration path without manual intervention
- **Impact:** Premature data loss perceived as bug; no backward-compatible format evolution
- **Fix:** Store ciphertext version prefix (e.g., 1-byte version prepended to cleartext before sealing); on decode, check version and route through appropriate decoder
- **Test:** Encode session with version=1, decode with version check; then simulate version mismatch and verify proper error not "malformedCiphertext"

### [RACE-09] CAPTRuntimeClient: `makeCommandEnvelope` collision risk — same op/payload → same commandID
- **Severity:** P3  | **Confidence:** medium  | **File:** CAPTRuntimeClient.swift:163-178
- **Evidence:** `makeCommandEnvelope` computes `digest = SHA256(seed).prefix(16)` where `seed = ["op": op, "payload": payload]`. Identical `op`+`payload` produces identical `commandID`. No sequence number or per-connection nonce
- **Mechanism:** Server receives duplicate `commandId`; idempotency key may not be sufficient if server keys off `commandId`
- **Impact:** Server may suppre...
