# Hermes ExecutionDriver Analysis Artifact

DriverRunId: dr-model-b1d5418ae26a5f16242278ad
Target: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs
ExternalPid: 39971
ExitCode: 0
ElapsedSeconds: 226.62

## Untrusted runtime output

# CAPT Swift/macOS Native App Audit Report

## COVERAGE
Inspected all Swift source files in:
- `/Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTNativeMac/` (App, Stores, Views)
- `/Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTCoreDesktop/` (Core Desktop components)
- Supporting files: `Package.swift`, `README.md`

## NO-FINDING
High-risk areas checked with no defects identified:
- **Code signing/Hardened runtime**: Build scripts (`script/build_and_run.sh`) enforce signing with fallback to ad-hoc; no runtime entitlement bypasses observed.
- **API evolution/backward compatibility**: Operator projection layer cleanly separates UI from runtime contracts; no version skew detected.
- **Packaging/install/update**: `script/install_local_runtime.sh` creates isolated venv; update mechanism relies on RuntimeService authority.

## OBSERVATIONS

### [ISSUE-001] Main-Actor Violations in State Updates
**ID**: GOV-MAINACTOR-001  
**Severity**: P0  
**Confidence**: High  
**Evidence**: 
- `CAPTOperatorStore.swift:150`: `connectionState = .connecting` (background Task)
- `CAPTOperatorStore.swift:158`: `connectionState = .connected` (background Task)
- `CAPTOperatorStore.swift:163`: `connectionState = .failed(message)` (background Task)
- Multiple similar updates to `@Published` properties (`runtimeIdentity`, `taskState`, `isBusy`, `lastError`) from `Task` blocks without `MainActor` confinement.
**Mechanism**: Store class is `@MainActor`, but `Task{}` executes on global concurrent executor. UI updates from non-main thread cause undefined behavior (missed updates, crashes).
**Impact**: UI may not reflect connection state, approval states, or errors; potential race conditions.
**Minimal Fix**: Wrap all `@Published` property assignments in `Task { @MainActor in ... }` or use `MainActor.run`.
**Test**: Simulate rapid connection/disconnection cycles; verify UI updates match state changes and no console warnings.

### [ISSUE-002] Chat Workspace Updates Not Published
**ID**: GOV-PUBLISH-002  
**Severity**: P0  
**Confidence**: High  
**Evidence**:
- `CAPTOperatorStore.swift:482`: `mutateWorkspace` updates `chatWorkspace` but sends no `objectWillChange`.
- `CAPTOperatorStore.swift:550`: `saveSessions` called after mutation but does not publish.
- `ChatView.swift:12`: `ForEach(store.messages)` relies on computed property with no publishing mechanism.
**Mechanism**: `chatWorkspace` is private mutable state; views depend on computed properties (`messages`, `pendingApproval`) that change when `chatWorkspace` mutates, but store lacks `objectWillChange` notification.
**Impact**: Chat view does not scroll to new messages; approval cards fail to appear; session restoration UI inconsistencies.
**Minimal Fix**: Add `@Published var chatWorkspace: CAPTNativeChatWorkspace` and update via `$chatWorkspace.wrappedValue = ...` or call `objectWillChange.send()` in `mutateWorkspace`.
**Test**: Send a message; verify message appears in chat view and auto-scroll triggers.

### [ISSUE-003] Unbounded Chat History Memory Growth
**ID**: GOV-MEMGROWTH-003  
**Severity**: P2  
**Confidence**: High  
**Evidence**:
- `CAPTNativeChatWorkspace.swift`: `sessions` array grows indefinitely; no pruning logic.
- `CAPTOperatorStore.swift:550`: `saveSessions` archives all sessions/messages to encrypted file.
**Mechanism**: No limit on retained chat sessions or message count; session store accumulates all history.
**Impact**: Memory usage increases linearly with session length; encrypted session file grows without bound, impacting launch time and disk usage.
**Minimal Fix**: Implement rotating history (e.g., retain last 500 messages per session) in `mergeRestoredSessions` and `newChat`.
**Test**: Simulate 10-hour chat session; verify memory usage plateaus and session file size stabilizes.

### [ISSUE-004] Error State Not Presented to User
**ID**: GOV-UXERROR-004  
**Severity**: P2  
**Confidence**: High  
**Evidence**:
- `CAPTOperatorStore.swift`: `@Published var lastError: String?` updated in 12+ locations.
- Zero UI references to `lastError` in any View file.
**Mechanism**: Errors logged internally but never surfaced via alert, banner, or inline message.
**Impact**: Users unaware of failures (connection, approval, execution); poor diagnosability.
**Minimal Fix**: Add `.alert(store.lastError, isPresented: .constant(store.lastError != nil))` to `ContentView` or `ChatView`.
**Test**: Trigger connection failure; verify user-visible error appears.

### [ISSUE-005] Accessibility Deficits
**ID**: GOV-ACCESS-005  
**Severity**: P2  
**Confidence**: Medium  
**Evidence**:
- `MessageRow.swift`: No `.accessibilityLabel()` on role/badge/text elements.
- `ApprovalCard.swift`: Buttons lack `.accessibilityHint()`; no dynamic type scaling.
- `ChatView.swift`: ScrollView accessibility not enhanced for screen readers.
**Mechanism**: Custom views omit standard accessibility traits; reliance on default SwiftUI heuristics insufficient.
**Impact**: VoiceOver users cannot discern message authorship, approval action purpose, or chat navigation state.
**Minimal Fix**: Add explicit labels (`Text(message.role.rawValue).accessibilityLabel("Role: \(message.role.rawValue)"`)), hints, and adjust hit testing.
**Test**: Validate with VoiceOver; all interactive elements announce purpose and state.

### [ISSUE-006] Incomplete Task Cancellation
**ID**: GOV-TASKCANCEL-006  
**Severity**: P3  
**Confidence**: Medium  
**Evidence**:
- `CAPTOperatorStore.swift`: Multiple `Task{}` blocks (e.g., `refreshIdentity`, `approvePending`) lack `Task.isCancelled` checks post-`await`.
- `ChatView.swift:62`: `.task(id: store.pendingApproval?.requestID)` tied to requestID but not view lifecycle.
**Mechanism**: Long-running tasks may complete after view dismissal or superseded request, causing stale state updates.
**Impact**: Wasted computation, potential UI flashes (e.g., expired approval shown after new chat).
**Minimal Fix**: Add `guard !Task.isCancelled else { return }` after each `await` in background tasks.
**Test**: Rapidly switch chats during approval flow; verify no stale approval cards appear.

### [ISSUE-007] Missing Automatic Runtime Reconnection
**ID**: GOV-RECONNECT-007  
**Severity**: P2  
**Confidence**: Medium  
**Evidence**:
- `CAPTOperatorStore.swift:170`: `connectionState = .disconnected` only on explicit `shutdown()` or `disconnect()`.
- No retry logic in `connect()` failure path; user must manually trigger reconnect.
**Mechanism**: On network/runtime hiccup, app enters `.failed` state requiring manual intervention.
**Impact**: Poor resilience in unstable environments; user-perceived fragility.
**Minimal Fix**: Implement exponential backoff reconnect in `handleGlobal` or connection failure handler.
**Test**: Kill RuntimeService process; verify automatic reconnection within 30s.

## SUMMARY
Critical threading model defects (ISSUE-001, ISSUE-002) undermine core UI correctness. Memory growth (ISSUE-003) and error reporting (ISSUE-004) affect long-term usability. Accessibility (ISSUE-005) and task cancellation (ISSUE-006) are secondary but notable. Recommend prioritizing MAINACTOR and PUBLISH fixes for immediate stability. No evidence of authority duplication or governance bypass; client correctly defers to RuntimeService. All observations derived from static analysis; dynamic validation recommended per work order constraints.
