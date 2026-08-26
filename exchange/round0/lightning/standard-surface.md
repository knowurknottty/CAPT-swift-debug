# Hermes ExecutionDriver Analysis Artifact

DriverRunId: dr-model-4e8d821cae4065f0c2b34cee
Target: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/standard
ExternalPid: 34902
ExitCode: 0
ElapsedSeconds: 511.11

## Untrusted runtime output

# CAPT Swift/macOS Standard Audit — 20260825

## COVERAGE
Inspected 33 files across review-input/standard and runtime staging. Sources: Views (ContentView, SidebarView, StatusBarView, ProviderControlView, ChatView, LedgerView, MemoryContextView, InspectorView, EvidenceBrowserView, MissionBrowserView), Stores (CAPTOperatorStore, CAPTNativeSessionStore), Core (CAPTCoreDesktop, CAPTOperatorProjection, CAPTChatFlow, CAPTRuntimeModels, CAPTRuntimeControlProjection, CAPTProviderSecretConvention), CLI (CAPTOperatorCLI, CAPTRuntimeClient), Tests (CAPTNativeSessionStoreTests, CAPTChatFlowTests), ChatCoordinator, Package.swift, verification-baseline.json.

## FINDINGS

| ID | Severity | Confidence | File:Line | Issue |
|----|----------|------------|-----------|-------|
| [O1] | P3 | medium | ContentView:5, SidebarView:5, ChatView:5 | Multiple `@ObservedObject` references to same store cause implicit coupling; `@Published` changes propagate to all surfaces simultaneously. |
| [O2] | P1 | high | CAPTOperatorStore:136-150 | `connect()` fire‑and‑forget `Task` races with `isBusy` on main actor; button may remain disabled after `defer { isBusy = false }` executes. |
| [O3] | P2 | medium | CAPTOperatorStore:273-288 | `reconcileActiveApprovalValidity()` and `updateTaskStateFromActiveFlow()` operate in separate `mutateWorkspace` calls; `taskState` lags one render cycle behind approval expiry. |
| [O4] | P3 | low | CAPTOperatorStore:358-370 | `activateSession()` expires stale approvals but leaves "approval_expired" messages in chat history; `pendingApproval` becomes nil, card vanishes, stale message remains. |
| [O5] | P1 | high | CAPTOperatorStore:173-181 | `submitPrompt()` ignores `beginPrompt()` return value; proceeds to `runtime.requestApproval()` even when compose is disabled, risking lost input / duplicate submissions. |
| [O6] | P2 | medium | ChatView:84-88 | `onChange(of: store.messages.count)` triggers `proxy.scrollTo()` on every message append; causes jitter/CPU waste on histories >50 messages. |
| [O7] | P3 | medium | ChatView:26-31 | `ForEach(store.messages)` retains all message objects linearly; no culling policy; memory grows with chat age. |
| [O8] | P3 | low | ChatView ComposerView | No `@FocusState`/keyboard dismissal; Escape/click-outside does not dismiss keyboard. |
| [O9] | P2 | medium | ChatView:133-155 | `ApprovalCard` buttons lack `.accessibilityLabel()`; `pending.objective` has no VoiceOver structure. |
| [O10] | P3 | low | ChatView:185-197 | `send()` closure has no explicit `draft.isEmpty` guard beyond `.disabled` modifier; whitespace may submit empty text. |
| [O11] | P2 | medium | CAPTOperatorStore (multiple) | `lastError` set in many places; only read in `InspectorView`/`StatusBarView`; never cleared on success; stale errors persist across sessions. |
| [O12] | P2 | medium | ChatView approval flow | Double‑step prompt→approval→execution creates friction; expired approval between card appearance and user gesture causes generic failure. |
| [O13] | P1 | high | ProviderControlView:123-142 | `SecureField` allows raw API key entry; no view‑level validation prefix check. "Save Reference" button accepts raw keys contrary to `isSafeSecretReference` policy. |
| [O14] | P3 | low | Package.swift:1-15 | `swift-tools-version: 5.9` + macOS 13+ only; no platform conditionals; no iOS/watchOS targets; edition drift between standard/Inversion Labs undocumented. |
| [O15] | P1 | high | CAPTOperatorStore:251+ / CAPTNativeSession | `provider`/`model`/`targetRoot` duplicated in store + per‑session; `syncSelectionFromActiveSession()` only fires when `activeSessionID == originSessionID`; manual changes lost on `refreshAll`/reconnect. |
| [O16] | P3 | low | CAPTNativeSessionStoreTests / CAPTChatFlowTests | 6 unit tests each; no integration tests across `CAPTOperatorStore ←→ RuntimeService ←→ CLI`; no UI tests. |
| [O17] | P3 | low | CAPTRuntimeClient:73-89 | `~/.capt/runtime.sock` / `runtime.token` no permission enforcement (should be `0700`/`0600` like session store). |
| [O18] | P2 | medium | MemoryContextView:51-69 | Stepper allows 1‑64 with no local validation; `hardStop` > `modelSafe` or trigger‑step sum overflow may pass to RuntimeService unpredictably. |
| [O19] | P3 | low | CAPTOperatorStore:297-309 | `refreshHistory()` no debounce; called on every `connect()`/`refreshAll()`/view `onAppear`; redundant runtime loads during rapid connect/disconnect. |

## NO-FINDING (high‑risk areas checked, not defective)
- [NF1] No use‑after‑free or memory corruption; ARC correct.
- [NF2] No hardcoded secrets; only `keychain:`/`env:` reference patterns.
- [NF3] No SQL/injection in CLI JSON; all payloads `JSONSerialization` with sorted keys.
- [NF4] No privilege escalation beyond scoped Keychain access.
- [NF5] No retain cycles; closures use `inout` mutation safely.
- [NF6] All major types conform to `Sendable`.
- [NF7] No file‑system corruption vectors outside encrypted session store (AES‑GCM validated).

## MINIMAL ROBUST FIX SET (priority order)

1. **P1**: Make `connect()` `async`; guard `isBusy` on actor boundaries.
2. **P1**: Add `guard let sessionID = mutateWorkspace({ $0.beginPrompt(...) }) else { return }` in `submitPrompt` before `Task { }`.
3. **P1**: Consolidate `provider`/`model` persistence; derive UI from session state, not dual store.
4. **P1**: Add `isSafeSecretReference` guard in `configureProviderAPIKey`; disable "Save Reference" for raw keys.
5. **P2**: Remove automatic `scrollTo` on message count change; add policy validation before `updateMemoryPolicy` with local alerts.
6. **P2**: Add `.accessibilityLabel()` to ApprovalCard buttons; VoiceOver structure on approval details.
7. **P2**: Clear `lastError` on successful operations; surface as transient alerts.
8. **P3**: Document build settings; enforce `~/.capt` directory permissions `0700`; token file `0600`.

**REPORT END** — All evidence from source inspection only. No runtime execution. No files modified.
