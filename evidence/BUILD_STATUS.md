# Swift build / sanitizer status

- Standard frozen installed-source snapshot: `swift test` PASS (64 tests, 7 gated skips), release build PASS, strict-concurrency build PASS, serialized TSan PASS, ASan PASS.
- Labs frozen installed-source snapshot: `swift test` PASS (64 tests, 4 gated skips), release build PASS, strict-concurrency build PASS, TSan PASS, ASan PASS.
- The first Standard TSan failure was a concurrent index-store temporary-file collision; serialized rerun is authoritative and passed.
- Live-runtime acceptance is tracked separately under `evidence/live/` and is not inferred from unit/sanitizer results.
