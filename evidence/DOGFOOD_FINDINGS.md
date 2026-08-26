# CAPT dogfood findings encountered during this audit

## DOGFOOD-001 — approval digest changes on trailing objective whitespace
`request_model_prompt_approval()` strips the objective before binding it. The approved execution path hashes the supplied objective without applying the same normalization. A one-line objective ending in `\n` gets an approved receipt and then deterministically fails at execution with `MODEL_PROMPT_APPROVAL_DIGEST_MISMATCH`.

Evidence: `capt-trailing-whitespace-approval-probe.json`.

Impact: non-canonical clients can mint apparently valid approvals that can never dispatch. The native Swift store currently trims user text before approval, so this is primarily a CAPT cross-surface contract defect rather than an authorization bypass.

Fix: define one canonical objective normalization function and apply it identically at approval and execution, or bind exact raw bytes with no normalization on either side. Add whitespace matrix contract tests.
