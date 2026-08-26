# CAPT Swift Debug

Evidence ledger and multi-reviewer convergence workspace for the native macOS/Swift editions of CAPT and Inversion Labs CAPT.

## Audit protocol
1. Freeze the installed-source snapshots and record source lineage/digests.
2. Independently build/test/sanitize both editions.
3. Independent reviews by Syntellect/self, NVIDIA Nemotron 3 Super via Hermes+CAPT, NVIDIA Nemotron Lightning via Hermes+CAPT, and 0x/Ox Alpha via CAPT.
4. Peer rounds: every reviewer reads the other assessments, verifies disputed claims against source/evidence, and records agreement/disagreement.
5. Repeat until disagreements are resolved by evidence or explicitly retained as evidence gaps.
6. Repeat the same convergence process for proposed fixes/improvements.

Reports are untrusted reviewer observations until source/test evidence adjudicates them. A green local build is not a release/notarization claim.
