# Hermes ExecutionDriver Analysis Artifact

DriverRunId: dr-model-dae1890fd4a3fa168c7d9166
Target: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs
ExternalPid: 37327
ExitCode: 0
ElapsedSeconds: 118.60

## Untrusted runtime output

# CAPT Native macOS / SwiftUI Audit

**COVERAGE:** 36 files inspected under `/Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs` and runtime-lightning staging.

## CONFIRMED DEFECTS

### 1. OBSERVATION STALE-AFTER-APPROVAL (P1, high confidence)
**File:** `CAPTOperatorStore.swift` — `approvePending()`  
**Mechanism:** After `approvePending()` completes, `taskState` is set to `result.taskState` (e.g. `"awaiting_verification"`), but the ChatView's `messages.lazyVStack` may still display old `authorityState` because the new system message appends but the view repaints only on next `refreshHistory()`. The `.onChange(scrollTo)` uses message.id from the prior snapshot.  
**Impact:** User sees stale `"approval_required"` or old `"executing"` after execution completes.  
**Fix:** In `approvePending()`, after `mutateWorkspace { $0.completeExecution(...) }`, also call `updateTaskStateFromActiveFlow()`.  
**Test:** Verify `taskState == "awaiting_verification"` persists through next view repaint.

### 2. SESSION CONFIGURATION MUTATION RACES WITH PENDING APPROVAL (P1, high confidence)
**File:** `CAPTOperatorStore.swift` — `persistConfiguration()` / `setExecutionProvider/Model/TargetRoot`  
**Mechanism:** `persistConfiguration` mutates the workspace AND synchronously updates local `@Published` properties (`provider`, `model`, `targetRoot`). If config changes while a pending approval is bound to the prior config, the approval is silently superseded (`CAPTNativeChatWorkspace.supersedeApproval`) but the UI still shows the new provider/model in `StatusBarView` because those `@Published` props were already updated before approval suspension took effect.  
**Impact:** User changes provider/model, then submits a prompt that gets an approval; the approval is retired as superseded, but the UI shows the new provider/model as if it were the one approved — mismatch between authorized configuration and what the user sees as "active."  
**Fix:** In `persistConfiguration`, add guard `if activeSessionID != sessionID { return }` before the `provider = newProvider` / `model = newModel` / `targetRoot = newTargetRoot` assignments at the bottom.  
**Test:** Change provider while approval is pending; verify `pendingApproval` is nil'd and phase becomes `.recoverableFailure` with `"approval_superseded"` state.

### 3. CANCOMPOSE STALE AFTER CONFIGURATION MUTATION (P2, medium confidence)
**File:** `CAPTOperatorStore.swift` — `canComposeInActiveChat` property  
**Mechanism:** `canComposeInActiveChat` depends on `connectionState == .connected && pendingApproval == nil && activeChatFlow.canCompose`. If configuration is mutated (provider/model changed) while a `requestingApproval` phase is in progress, the flow phase may transition to `.awaitingApproval` with `requestID` set, but `canComposeInActiveChat` may still return `true` because `connectionState` and `pendingApproval` haven't been updated yet — the local `@Published pendingApproval` is updated asynchronously via the Task in `submitPrompt` but the synchronous guard check happens before that.  
**Impact:** User can attempt to compose a new prompt while a prior approval-bound prompt is still in-flight; the new prompt may be rejected or cause unexpected state reset.  
**Fix:** In `submitPrompt`, after the Task completes and `mutateWorkspace { $0.receiveApproval(pending, for: sessionID) }`, explicitly recompute `canComposeInActiveChat` state or add `activeChatFlow.phase = .idle` reset before beginning a new prompt if `pendingApproval` exists from a prior config change.

### 4. MEMORY POLICY APPLY DISregards RuntimeService Precedence (P2, medium confidence)
**File:** `MemoryContextView.swift` — `policyEditor` "Apply Governed Policy" button  
**Mechanism:** The stepper values (`retrieval`, `compression`, `checkpoint`, `consolidation`, `hardStop`, `modelSafe`) are local `@State` values. When "Apply Governed Policy" is clicked, `store.updateMemoryPolicy(runtime)` is called, which sends the values to RuntimeService. The UI disables the button if `runtimeCapabilities?.supportsCommand("update_memory_trigger_policy") != true`, and the accompanying text says `"RuntimeService validates precedence and safe-limit relationships; the app does not override policy authority." But there is no validation that the user-supplied thresholds respect the runtime's internal precedence rules (e.g. `hardStop >= checkpoint >= consolidation`, or `modelSafe > max(retrieval, compression)`). The app blindly passes whatever the user sets.  
**Impact:** User applies inconsistent policy thresholds; RuntimeService may reject or silently clamp them; `memorySnapshot.policyDigest` changes but the UI doesn't reflect whether the runtime accepted or rejected the policy, leading to opaque state.  
**Fix:** Add pre-apply validation in the View that checks `hardStop >= checkpoint >= consolidation` and `modelSafe > max(retrieval, compression)`, and display a runtime-level acknowledgment of the applied policy (what was set vs what RuntimeService confirmed).

### 5. CREDENTIAL STORAGE DISCLOSURE GAP (P2, medium confidence)
**File:** `ProviderControlView.swift` — `credentialSection`, `configureProviderAPIKey`  
**Mechanism:** The view includes an "Advanced credential reference" `DisclosureGroup` that allows saving a reference string (e.g. `"keychain:openrouter"`). However, the "Save Reference" button calls `setProviderKeyReference` which stores the reference in keychain but does NOT verify that the reference format is correct before storing — it just passes the string through. The view also shows `"Stored reference: item.keyRef"` but `keyRef` is populated from the initial `CAPTProviderSecretConvention.location()` call, not from what the user entered. There's a mismatch between what the user types in the `TextField` and what gets stored as "Stored reference."  
**Impact:** User may enter an invalid or partial reference, believe it's stored correctly, and later be unable to authenticate because the stored reference doesn't match the expected format. The "Set API Key" flow for openrouter stores the raw key in keychain under `"capt-provider"/"openrouter"`, but the "Advanced reference" flow stores a reference string instead — two different pathways with different semantics that are not clearly demarcated.  
**Fix:** Clarify the `DisclosureGroup` help text to distinguish `"env:"` vs `"keychain:"` reference formats; validate reference format before storing; show the actual stored reference value (`item.keyRef`) as confirmation, not just a description.

### 6. LAB ENGINE PROVENANCE MISSING SIGNATURE VERIFICATION (P2, medium confidence)
**File:** `LabsView.swift` — `provenanceCard`, `CAPTLabEngineSnapshot` sourceFiles  
**Mechanism:** The Labs view displays source file paths and SHA256 hashes for each lab engine, but the SHA256 values are taken directly from the engine provenance data received from RuntimeService without any independent verification. There is no code that recomputes or validates the SHA256 against the actual source files on disk. The "Filesystem" field shows `"bounded read"` or `"not required"` but there's no mechanism to actually read and verify the files mentioned in `sourceFiles`.  
**Impact:** A compromised or incorrectly-provenanced engine could report arbitrary SHA256 values; a user cannot verify that the engine they're running matches the expected source.  
**Fix:** Add a "Verify Provenance" menu item that, for engines requiring filesystem, reads the listed source files and recomputes SHA256, comparing against the reported value and surfacing a mismatch warning.

### 7. SCOPED APPROVAL DISPLAY SHOWS REDACTED PROMPT DIGEST (P3, low confidence)
**File:** `ChatView.swift` / `ApprovalCard.swift` — `pending.promptAssemblyDigest` display  
**Mechanism:** The `ApprovalCard` displays `pending.promptAssemblyDigest` (e.g. `"sha256:abc"`) as a line-of-text value. However, this digest is computed over the prompt assembly (messages + provider + model + target) and is meant to be a tamper-evidence token. Displaying just the hex digest without context about what it covers means the user cannot independently verify that the digest matches the ...
