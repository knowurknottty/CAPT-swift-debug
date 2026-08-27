# Packet01 frontend dogfood evidence — 2026-08-27

This directory mirrors the review-safe evidence captured during the Inversion Labs CAPT frontend dogfood pass.

Included:
- `DOGFOOD_LEDGER.md`
- `glm-plan-prompt.txt`
- `frontend.log` and `frontend-isolated.log` (currently zero-byte preserved artifacts)
- every screenshot whose numeric prefix is `001` through `030`, inclusive
- `MANIFEST.json` with byte sizes and SHA-256 digests

Intentionally excluded from this **public** repository:
- `state/` (`runtime.db`, token/socket/PID files, runtime venv, etc.)
- `Inversion Labs CAPT Packet01.app/`

The screenshots are primary visual evidence. The ledger identifies the known frontend/runtime-coherence defects and maps them to screenshot IDs.
