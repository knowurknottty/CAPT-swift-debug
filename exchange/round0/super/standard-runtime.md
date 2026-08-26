# Hermes ExecutionDriver Analysis Artifact

DriverRunId: dr-model-55ea8cf6c06beab6cbaabf9a
Target: /Users/knowurknot/.capt/reviews/capt-swift-full-audit-20260825/review-input/standard
ExternalPid: 39208
ExitCode: 0
ElapsedSeconds: 143.78

## Untrusted runtime output

# CAPT Swift/macOS Edition Audit Observations

## COVERAGE
Inspected files:
- Sources/CAPTNativeMac/Services/CAPTBackgroundRuntime.swift
- Sources/CAPTCoreDesktop/CAPTNativeSessionStore.swift
- Sources/CAPTCoreDesktop/CAPTRuntimeClient.swift
- Sources/CAPTNativeMac/App/CAPTNativeMacApp.swift
- runtime-super/staging/dr-model-55ea8cf6c06beab6cbaabf9a/verification-baseline.json
- Sampled other Swift files in Sources/ and Tests/ for cross-reference (no mutations).

## OBSERVATIONS

### ID: CAPT-SWIFT-001
**Severity**: P2  
**Confidence**: High  
**File**: Sources/CAPTNativeDesktop/CAPTNativeSessionStore.swift:107-128  
**Mechanism**: The `save(_:)` method writes the encrypted session file atomically via `write(to:options: .atomic)`, then immediately updates file permissions to 0o600 via `setAttributes`. Between these two steps, the file exists with default umask-derived permissions (typically 0o644) on the filesystem, potentially exposing decryption keys or session data to other local users.  
**Impact**: Local information disclosure; session keys or decrypted session data could be read by other users on the same machine before permissions are tightened.  
**Fix**: Set file attributes *before* writing, or use a single atomic operation that includes permissions. On macOS, create the file with desired permissions initially via `FileManager.createFile(atPath:contents:attributes:)`.  
**Test**: Verify that immediately after `save()` completes, the file's permissions are 0o600 at all observable intervals (e.g., via a race-test tool that attempts to read the file during the save operation).

### ID: CAPT-SWIFT-002
**Severity**: P2  
**Confidence**: High  
**File**: Sources/CAPTNativeDesktop/CAPTNativeSessionStore.swift:4  
**Mechanism**: Class `CAPTEncryptedSessionStore` is marked `@unchecked Sendable`. While its stored properties (`fileURL: URL`, `keyProvider: any CAPTSessionKeyProviding`) are Sendable, the class lacks internal synchronization for concurrent access to `load()` and `save(_:)`. If multiple tasks invoke these methods concurrently (e.g., via SwiftUI background tasks), file read/write races could occur, leading to corrupted session data or lost updates.  
**Impact**: Potential session data corruption, loss of pending approvals, or cryptographic errors due to interleaved reads/writes.  
**Fix**: Replace `@unchecked Sendable` with explicit synchronization (e.g., make the class an `actor`, or use a `NSLock`/`DispatchQueue` to serialize file access).  
**Test**: Launch multiple concurrent tasks calling `load()` and `save()` with overlapping file assertions; verify no data corruption or crashes occur.

### ID: CAPT-SWIFT-003
**Severity**: P1  
**Confidence**: High  
**File**: Sources/CAPTCoreDesktop/CAPTRuntimeClient.swift:59-76  
**Mechanism**: In `connect()`, if the token file is missing or empty, the function throws `CAPTRuntimeClientError.tokenMissing` *before* calling `disconnect()`. This leaves any existing socket connection open (socketFD not reset), causing a resource leak. Subsequent connection attempts may fail due to "address already in use" or leave stale state.  
**Impact**: Socket file descriptor leak; potential denial-of-service if repeated token-missing events exhaust available file descriptors.  
**Fix**: Call `disconnect()` unconditionally at the start of `connect()` (before token validation) to ensure a clean state.  
**Test**: Simulate missing token file; verify no open socket remains after the thrown error (via `lsof` or checking socketFD state).

### ID: CAPT-SWIFT-004
**Severity**: P1  
**Confidence**: High  
**File**: Sources/CAPTNativeMac/Services/CAPTBackgroundRuntime.swift:209-221  
**Mechanism**: In `shutdown()`, if `client.command(op: "shutdown")` throws (e.g., due to network timeout or runtime unresponsiveness), the function propagates the error *without* calling `client.disconnect()`. This leaves the socket open and the client in an authenticated-but-orphaned state.  
**Impact**: Socket leak; potential inability to reconnect due to stale socket or authentication state confusion.  
**Fix**: Ensure `client.disconnect()` is called in a `defer` block or `catch` clause before propagating the error.  
**Test**: Mock `client.command` to throw; verify socket is closed after `shutdown()` error.

### ID: CAPT-SWIFT-005
**Severity**: P2  
**Confidence**: High  
**File**: Sources/CAPTNativeMac/Services/CAPTBackgroundRuntime.swift:179-190, 192-203  
**Mechanism**: Both `cancelTask(_:)` and `cancelDriverRun(_:)` mirror the flaw in CAPT-SWIFT-004: if the underlying `client.command` throws, `client.disconnect()` is not called, leaving the socket open.  
**Impact**: Socket leak during cancellation error paths; cumulative resource exhaustion.  
**Fix**: Apply same fix as CAPT-SWIFT-004: guarantee `client.disconnect()` on error via `defer` or `catch`.  
**Test**: Mock command to throw in cancellation methods; verify socket closure.

### ID: CAPT-SWIFT-006
**Severity**: P3  
**Confidence**: Medium  
**File**: Sources/CAPTCoreDesktop/CAPTRuntimeClient.swift:206-215  
**Mechanism**: The `receiveUnlocked()` method uses a loop to read exact byte counts via `readExact()`. If the remote peer sends partial frames slowly (or stalls mid-frame), this loop will block indefinitely on `recv()`, potentially hanging the calling thread. While the caller holds an `NSLock`, this could stall all client operations if invoked from the actor.  
**Impact**: Denial-of-service via network stalling; no timeout mechanism for socket reads.  
**Fix**: Implement configurable read timeouts (e.g., via `setsockopt` with `SO_RCVTIMEO`) or use non-blocking I/O with select/poll.  
**Test**: Simulate delayed peer transmission; verify operation does not hang indefinitely.

### ID: CAPT-SWIFT-007
**Severity**: P3  
**Confidence**: Low  
**File**: Sources/CAPTNativeMac/Services/CAPTBackgroundRuntime.swift:4  
**Mechanism**: The `CAPTBackgroundRuntime` actor initializes synchronous dependencies (`CAPTRuntimeClient`, `CAPTChatCoordinator`, etc.) in its initializer. If any of these initializers perform blocking work (e.g., synchronous file I/O), they could delay actor initialization and block the thread that creates the actor.  
**Impact**: Potential UI thread blocking if actor is created on main thread (not observed in inspected code).  
**Fix**: Ensure initializers are non-blocking; offload heavy setup to `Task` if needed.  
**Test**: Verify initialization completes quickly (<10ms) under typical conditions.

## NO-FINDING (High-Risk Areas Checked)
- **Concurrency/Races in Actor Model**: All inspected asynchronous boundaries are properly actor-isolated (`CAPTBackgroundRuntime` as actor; `CAPTRuntimeClient` uses internal `NSLock`). No data races observed in inspected code paths.
- **Process Spawning/IPC**: No direct `fork()/exec()` observed; IPC via Unix sockets appears correctly framed with length prefixes and JSON validation.
- **Cryptography/Key Handling**: Session encryption uses AES-GCM with 32-bit keys from Keychain; key derivation and accessibility (`AfterFirstUnlockThisDeviceOnly`) are appropriate.
- **Filesystem Permissions**: Session store directory created with 0o700; file written atomically then chmod to 0o600 (except race noted in CAPT-SWIFT-001).
- **Approval Binding/Lifecycle**: Approval requests flow through `CAPTChatCoordinator` (not inspected in depth) but RPC idempotency keys use UUIDs; no obvious duplication risks.
- **Error Propagation**: Errors are properly typed and propagated; no swallowing of critical errors in inspected paths.
- **Crash/Restart Recovery**: Session store includes migration logic for legacy files; checkpoint/resume commands exist in runtime client.
- **TOCTOU**: No time-of-check/time-of-use vulnerabilities observed in file operations (aside from CAPT-SWIFT-001 permission race).
- **Authority Leakage**: No privilege escalation paths observed; operatorID/sessionID scoped to runtime client.

**Note**: All observations are based on static analysis of the provided snapshot. No dynamic testing or network calls were performed per work order constraints. Severity reflects potential impact assuming typical macOS multi-user environment.
