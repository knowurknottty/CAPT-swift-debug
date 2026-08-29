# CAPT Packet 01 Frontend Dogfood Ledger

Target: installed `/Users/knowurknot/Applications/Inversion Labs CAPT.app`
Canonical test PID at start: `20851` (PID may rotate; verify executable path before each interaction)
Sprint worktree: `/Users/knowurknot/capt_workspace/capt_core_packet01_20260827`
Branch: `sprint/packet01-context-mission-resolver-20260827`
Baseline: `3aee7370bac880aed99ce3c9ecfaa6d9ff48101e`

## Rules
- Drive implementation/review through the Inversion Labs CAPT frontend.
- GLM 5.3 Flash, Hy3, and MiMo-V2.5 must use OpenRouter.
- Screenshot every frontend/interface/state transition and correlate visual state with reported/runtime state.
- Do not trust a programmatic success report when the visual state disagrees.
- Do not mutate shared dirty CAPT checkouts.

## Issues / observations

### DF-P01-001 — screenshot target assumption hid CAPT window
Classification: dogfood harness/tooling assumption, not yet a CAPT defect.
Evidence: screenshots 001-003 captured display 1 while Labs window was located on display 2 at logical x=3008.
Resolution: capture display containing target CAPT window.

### DF-P01-002 — concurrent CAPTNativeMac processes create ambiguous visual targeting
Classification: environment/concurrency hazard; possible product identity UX issue.
Observed processes include installed Labs plus development-build CAPT apps from other worktrees.
Mitigation: resolve exact executable path + PID before every interaction; never infer target by process name alone.

### DF-P01-003 — installed Labs runtime launch failure shown in UI
Classification: product defect / runtime bootstrap failure, open.
Visual: Runtime state `Failed: CAPT runtime launch failed: capt start exited with status 1`; `Not connected`.
Evidence: screenshots 004, 005, 007, 009.

### DF-P01-004 — stale/invalid provider-model projection while runtime unavailable
Classification: product defect / state projection coherence, open.
Visual: provider `ollama` paired with model `stealth/ox-alpha` while disconnected, despite OpenRouter being required for remote stealth model.
Evidence: screenshots 004, 005, 007, 009.

### DF-P01-005 — keychain authorization interrupts normal frontend recovery
Classification: expected macOS security interaction unless recurrence proves excessive; monitor.
Visual: Labs requests access to `com.inversionlabs.capt.lab.native-session-cache` and blocks UI pending local password authorization.
Evidence: screenshot 009.

### DF-P01-006 — Connected indicator coexists with impossible frame-length error
Classification: product defect / runtime transport framing, high severity, open.
Visual: runtime `Connected` while inspector still projects `ollama / stealth/ox-alpha`; Last error `CAPT frame exceeds limit: 574234740 bytes`.
Evidence: screenshot 011.
Risk: green connectivity state can mask a corrupted/invalid runtime response frame; provider/model projection cannot be trusted until refreshed from authoritative runtime state.

### DF-P01-007 — OpenRouter ACTIVE does not control execution provider
Classification: product defect / provider authority projection, high severity, open.
Visual: Providers surface marks OpenRouter ACTIVE while inspector and status bar remain `ollama / qwen3.6-fable-fusion:latest`.
Action: visible OpenRouter `Use` control exercised; no visible convergence afterward.
Evidence: screenshots 016-017.

### DF-HARNESS-001 — global click coordinate focused browser instead of CAPT
Classification: dogfood harness/operator error, not CAPT product defect.
Action: attempted to focus CAPT Target root; path text landed in browser GPT editor instead.
Evidence: screenshot 022.
Mitigation: stop inferred global clicks for text entry; resolve exact CG display bounds and/or use PID-bound accessibility before further typing.

### DF-P01-008 — installed Labs app self-terminates after clean relaunch
Classification: product defect / native app lifecycle, high severity, open.
Observed: installed `/Users/knowurknot/Applications/Inversion Labs CAPT.app` relaunched as PID 38417, rendered correctly, then exited before the next accessibility bind.
Evidence: screenshot 023 plus PID disappearance.
Mitigation for sprint: use the dedicated uniquely-bundled Packet01 Labs frontend so other same-bundle dev windows cannot steal focus/evidence.
