# Hermes ExecutionDriver Analysis Artifact

DriverRunId: dr-model-07791585700249044a6a8fec
Target: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs
ExternalPid: 39593
ExitCode: 0
ElapsedSeconds: 543.92

## Untrusted runtime output

COVERAGE
Inspected: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTNativeMac/Services/CAPTBackgroundRuntime.swift, /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTCoreDesktop/CAPTChatCoordinator.swift, /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTNativeMac/Stores/CAPTEncryptedSessionStore.swift (via CAPTNativeSessionStore.swift), /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTNativeMac/Stores/CAPTOperatorStore.swift, /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTCoreDesktop/CAPTRuntimeClient.swift, /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/runtime-super/staging/dr-model-07791585700249044a6a8fec/verification-baseline.json

NO-FINDING
- Cryptography/key handling: AES-GCM usage with 32-byte key from Secure Enclave via Keychain appears correct.
- Socket/file descriptor handling: CAPTRuntimeClient properly closes socket on error and deinit.
- Authority leakage: No obvious privilege escalation in approval flow.
- Injection/path traversal: No user input used in file paths without sanitization (e.g., targetRoot is passed as string but not used in file operations in inspected code).
- Trust boundaries: XPC/unix socket authentication via token appears intact.

FINDINGS

CAPT-SWIFT-001
Severity: P2
Confidence: High
File: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTNativeMac/Services/CAPTBackgroundRuntime.swift:36-52
Mechanism: connect() retries immediately after bootstrapper.start() without checking runtime readiness. If bootstrapper succeeds but runtime is not yet accepting connections, the second connect() fails.
Impact: Connection failure during startup, requiring user to retry connect manually.
Fix: Implement retry with exponential backoff or readiness polling (e.g., loop with short delays and identity query) after bootstrapper start.
Test: Mock bootstrapper to succeed but delay runtime socket acceptance; verify connect() eventually succeeds after retries.

CAPT-SWIFT-002
Severity: P2
Confidence: High
File: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTCoreDesktop/CAPTChatCoordinator.swift:58-68
Mechanism: approveAndRun() queries task state immediately after approval decision without handling potential delay in task state stream creation.
Impact: Approved execution may fail to display task state if runtime lag occurs, showing error instead of running state.
Fix: Retry get_state query with timeout (e.g., 3 attempts over 2s) or return default state on first failure and log.
Test: Introduce artificial delay in runtime task state creation; verify approveAndRun() eventually retrieves state.

CAPT-SWIFT-003
Severity: P1
Confidence: Medium
File: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTNativeMac/Stores/CAPTEncryptedSessionStore.swift:78-90
Mechanism: load() calls keyProvider.keyData() which may throw if Keychain is locked (pre-unlock on macOS). This prevents session decryption and causes startup failure.
Impact: Users cannot restore prior sessions after reboot if app launches before Keychain unlock (e.g., auto-login scenarios).
Fix: Catch keychain errors and fallback to empty session list with warning, or defer load until Keychain ready via observer.
Test: Simulate locked Keychain state; verify app launches with empty sessions and logs warning.

CAPT-SWIFT-004
Severity: P2
Confidence: Medium
File: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/labs/Sources/CAPTNativeMac/Services/CAPTBackgroundRuntime.swift:145-152
Mechanism: shutdown() calls client.command(op: "shutdown") then unconditionally disconnects. If shutdown command fails (e.g., runtime unresponsive), client disconnects leaving runtime running.
Impact: User believes runtime is shut down but it remains active, consuming resources.
Fix: Only disconnect on successful shutdown command; on failure, keep connection open and surface error.
Test: Mock client.command to throw error on shutdown; verify client remains connected and error is returned.
