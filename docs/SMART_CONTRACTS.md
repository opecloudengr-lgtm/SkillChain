# SkillChain Smart Contract Architecture — Security Audit & Revised Specification

**Reviewer role:** Senior Blockchain Security Auditor
**Scope:** Architecture-level review (no implementation code) of the modular RBAC + soulbound-certificate system on Electroneum Smart Chain (EVM-compatible).
**Method:** Static review of the provided specification against common EVM vulnerability classes, upgrade-safety patterns, and credentialing-system best practice.

---

## 0. Executive Summary

The architecture is directionally sound — modular separation of registries, RBAC, and a soulbound credential model are the right primitives for an institutional certificate system. However, the spec as written has several gaps that would become real vulnerabilities or operational failures at implementation time:

- **No single point of failure protection** on SUPER_ADMIN or the proxy admin — both are described as single roles, not multi-sig/timelocked entities, despite a "Timelock DAO" appearing in the diagram with no defined authority.
- **No wallet-loss or compromise recovery path** for students or institutions — a critical gap for a soulbound (non-transferable) credential system, since "non-transferable" normally also means "non-recoverable" unless explicitly designed otherwise.
- **Reentrancy guards are named but not scoped to actual external call sites** — the spec doesn't identify *where* external calls happen (metadata resolution, signature verification callbacks), so "apply nonReentrant" is directionally correct but currently unverifiable.
- **Integer/timestamp validation is implied, not specified** — no explicit invariants for `expiryDate > issueDate`, counter bounds, or hash length.
- **Event design lacks indexing strategy, versioning, and query-support fields**, which will matter a lot for a verification-heavy public product.
- **Emergency pause is binary (global on/off)** with no per-institution circuit breaker, which is a large blast-radius design for a multi-tenant registry.
- **No explicit token standard** is named for the "NFT Certificate" — this materially affects transfer-blocking correctness, tooling compatibility, and audit surface.

Below: findings by category (Critical → Low), followed by a revised specification incorporating fixes.

---

## 1. Access Control

**Findings**

| Severity | Issue |
|---|---|
| Critical | `SUPER_ADMIN` is described as a role but not bound to a multi-sig or the "Timelock DAO" shown in the architecture diagram. As written, a single compromised key can upgrade contracts, pause the system, and reassign all roles. |
| High | The role hierarchy (`SUPER_ADMIN → ADMIN → INSTITUTION → ISSUER → AUDITOR → PUBLIC`) is drawn as linear inheritance, but the permissions table shows non-overlapping capabilities (e.g., ADMIN cannot issue, INSTITUTION cannot approve institutions). This should be modeled as **role sets with explicit grants**, not a strict hierarchy — a linear model risks accidental privilege inheritance in implementation if `onlyAdmin` is coded as "ADMIN or higher." |
| High | No separation between an **institution's identity wallet** and its **operational issuer wallets**. If `INSTITUTION` manages `ISSUER` roles but the institution wallet itself can also issue, a compromised institution wallet has unbounded issuance power with no rate limiting. |
| Medium | No role for a **compliance/legal hold** function distinct from `AUDITOR` (read-only) and `ADMIN` (suspend). Certificate disputes (e.g., fraud investigations) need a role that can flag a certificate as "under review" without full revocation. |
| Medium | `AUDITOR` role's actual permissions aren't specified beyond "verify certificates" — but verification is listed elsewhere as public-readable. If `AUDITOR` has no exclusive capability, the role is redundant; if it does (e.g., viewing revocation reasons or institution compliance history), that must be specified. |
| Low | No mention of **role expiry** — institution/issuer approvals appear permanent until explicitly suspended, with no periodic re-attestation requirement. |

**Recommendation:** Model roles as an explicit capability matrix (see §Revised Spec), require multi-sig + timelock for `SUPER_ADMIN`, and separate institution identity from issuer operational keys.

---

## 2. Reentrancy Risks

**Findings**

- The spec correctly lists `nonReentrant` for issue/revoke/upgrade, but doesn't identify the **external call boundary**. In this system the realistic reentrancy vectors are:
  - A call into `MetadataRegistry` during `issueCertificate()` if that registry is a separate deployed contract (cross-contract call = external call, even without ETH transfer).
  - Any future integration where `issueCertificate()` triggers a callback (e.g., an ERC-721 `onERC721Received` hook, if the soulbound token is implemented on top of ERC-721 with transfer restrictions rather than a non-hooked custom token).
- Checks-Effects-Interactions is stated as "always enforced" but the storage-write flow in §17 (Issuance Flow) shows `CertificateManager → CertificateRegistry (store) → MetadataRegistry (store) → mint → emit event`. If `MetadataRegistry` write happens *before* the certificate is marked as existing in `CertificateRegistry`, a malicious metadata resolver could theoretically re-enter and reference a certificate ID that isn't yet finalized. The order of effects vs. interactions needs explicit sequencing, not just a principle statement.

**Recommendation:** Explicitly document the external-call boundary per function, and require: state finalized in `CertificateRegistry` **before** any cross-contract call to `MetadataRegistry` or any hook-triggering mint.

---

## 3. Integer Safety

**Findings**

| Severity | Issue |
|---|---|
| High | No explicit invariant that `expiryDate > issueDate`, or that `expiryDate == 0` is treated as "non-expiring" vs. "invalid." Ambiguous zero-values are a common source of logic errors in credential expiry checks. |
| Medium | `certificateCounter` / `institutionCounter` are unbounded incrementing integers with no specified type width. On EVM 0.8+, overflow reverts automatically, but the practical risk is **ID exhaustion or predictability**, not overflow — sequential IDs make certificate IDs guessable, which matters for a public verification system where enumeration could be used for scraping/social engineering. Recommend hash-derived or UUID-style opaque IDs (already listed as a stack component: UUID) rather than sequential counters, or use counters only as internal indices with a separate public-facing opaque identifier. |
| Medium | No specified max length / bounds for strings (`courseName`, `country`, `metadataURI`, `revokedReason`). Unbounded string storage is a gas-griefing and storage-bloat vector — a malicious or careless institution could submit multi-kilobyte strings. |
| Low | No explicit bounds check on `certificateHash` length (should be fixed-width, e.g., 32 bytes / bytes32, not arbitrary-length `bytes`). |

**Recommendation:** Add explicit invariants and bounds validation as first-class items in `_validateCertificate()` and `_validateInstitution()`, not left implicit.

---

## 4. Upgradeability

**Findings**

| Severity | Issue |
|---|---|
| Critical | Proxy Admin is not bound to the Timelock DAO or multi-sig in the spec — §15 shows `ImplementationV2 → ProxyAdmin` with no governance gate drawn. An unprotected `upgrade()` restricted only to "Super Admin" (a single EOA/role) is a critical centralization and rug risk for a credentialing system institutions and employers will rely on for trust. |
| High | "Storage layout remains append-only" is stated as a rule but not as an enforced check. Recommend a **storage-layout diff check as a mandatory CI gate** before any upgrade deployment (e.g., automated slot-diffing), not just a written convention. |
| Medium | No mention of **initializer protection** (i.e., ensuring the implementation contract itself cannot be initialized directly, and that `initialize()` can only run once). This is one of the most common transparent-proxy misconfiguration bugs. |
| Medium | No storage gaps specified for future variable additions in base/inherited contracts — needed to preserve append-only compatibility across upgrades when contracts use inheritance. |
| Low | Transparent Proxy Pattern is chosen without discussing the trade-off against UUPS (which moves upgrade logic into the implementation, reducing proxy gas overhead but requiring careful `_authorizeUpgrade` guarding). Given this system prioritizes safety over gas, Transparent Proxy is a reasonable choice — but the rationale should be documented for future maintainers rather than assumed. |

**Recommendation:** Bind `ProxyAdmin` to the Timelock DAO with a minimum delay (e.g., 48–72h) and multi-sig execution; add automated storage-layout verification to the deployment pipeline; explicitly require initializer-lock on implementation contracts.

---

## 5. Event Design

**Findings**

- No indexing strategy specified — e.g., `CertificateIssued` should index `institutionId`, `studentWallet`, and `certificateId` to support efficient off-chain querying (the primary use case for a public verification portal).
- No **event versioning**. If an upgrade changes the data emitted for `CertificateIssued`, indexers built against the old event will break silently. Recommend a `schemaVersion` field or clearly versioned event names across upgrades.
- No emitted event for **metadata updates** distinct from issuance — §4 (`MetadataRegistry`) allows "updates (where permitted)" but §5 (Events) has no `MetadataUpdated` event. Silent metadata mutation is itself an auditability risk for a credentialing product.
- No event for **role expiry / re-attestation**, if that recommendation from §1 is adopted.
- Missing a `CertificateStatusQueried` or similar off-chain analytics hook is fine to omit (verification is typically a `view` call and doesn't need an event) — but confirm `verifyCertificate()` is `view`/non-state-changing; if it's ever state-changing (e.g., to log verification counts on-chain), that needs its own event and gas consideration.

**Recommendation:** Add `MetadataUpdated`, `RoleExpired`, and a documented indexing/versioning convention for every event listed.

---

## 6. Gas Optimization

**Findings**

- "Packed Structs" and "Efficient Events" are listed as goals but no actual field ordering or bit-packing plan is given. E.g., in the Certificate struct: `status` flags (`revoked`, active/expired) could be packed into a single `uint8` bitmap alongside `institutionId` in one storage slot rather than separate `bool`/`uint256` fields.
- `issueDate`, `expiryDate`, `createdAt` could be packed as `uint40` (sufficient until year 2255) instead of full `uint256`, saving storage slots when combined with other sub-256-bit fields.
- Batch issuance is listed as a "future" capability — given institutions will likely issue certificates in cohorts (e.g., end of semester), this should be a **launch-time** feature, not deferred, since retrofitting batch operations after storage layout is fixed is more expensive under an append-only upgrade constraint.
- No mention of using `calldata` vs `memory` for read-only function parameters — minor but relevant for institutions submitting large metadata batches.

**Recommendation:** Specify struct packing layout explicitly in the storage section; move batch issuance into the initial release scope given the append-only upgrade constraint makes later restructuring costly.

---

## 7. Emergency Recovery

**Findings**

| Severity | Issue |
|---|---|
| Critical | Pause is **global only** — pausing halts all institutions' ability to issue/revoke, not just a compromised one. For a multi-tenant registry, a single institution's key compromise (e.g., a university's admin wallet phished) should not require freezing the entire network. Need a **per-institution suspend** capability that's faster and lower-blast-radius than global pause (this already exists as `suspendInstitution()`, but the spec doesn't clarify whether suspension takes effect immediately/atomically or requires a timelock — immediate action is needed for compromise response). |
| Critical | No **student wallet recovery path**. Since certificates are soulbound to a wallet address and standard transfers are disabled, a student who loses their private key permanently loses access to provable credentials, with no specified reissuance mechanism. This is a major product-level gap, not just a security nit. |
| High | No **compromised-issuer response flow** distinct from institution suspension — e.g., if a single `ISSUER` sub-account (not the institution wallet itself) is compromised, can the institution revoke just that issuer's role without institutional suspension? The permissions table implies yes ("INSTITUTION: Manage issuers") but no emergency-specific fast path is described. |
| Medium | No **guardian/social-recovery** or **admin-assisted reissuance** process defined for the wallet-loss case. |

**Recommendation:** Add a `RecoveryModule` (see revised architecture) supporting institution-attested certificate reissuance to a new student wallet, with the original certificate marked `superseded` (not deleted) for audit continuity. Add per-institution pause independent of global pause.

---

## 8. Certificate Ownership

**Findings**

- The spec calls the credential an "NFT Certificate" but never names a token standard. This matters:
  - If built on **ERC-721** with transfer functions simply reverting, that's a common and audit-friendly soulbound pattern, but must explicitly override `transferFrom`, `safeTransferFrom`, and `approve`/`setApprovalForAll` to revert, not just "disable" them informally — otherwise wallets/marketplaces that call these functions directly could hit unexpected states.
  - Alternatives like **ERC-5484** (Consensual Soulbound Tokens, which support a defined `burn` authority) or **ERC-4973** (Account-bound Tokens) are purpose-built for this use case and would reduce custom-code audit surface.
- "Institutions cannot reclaim ownership after issuance" is a good design principle, but combined with no wallet-recovery path (§7), this creates a permanent-loss scenario with no remediation. Recommend: institutions *can* issue a superseding certificate (new token, old one marked `superseded`) but never seize/transfer the original.
- No explicit statement on whether a `revoked` certificate is a permanent terminal state or can be reversed (e.g., revoked-in-error). §22 invariant testing states "Revoked certificates cannot return to a valid state" — if that's an intentional design choice (favoring auditability over correction), it should be paired with a `reissueCertificate()` flow so institutions have a compliant path to correct mistakes without violating the invariant.

**Recommendation:** Name the token standard explicitly (ERC-721 with hard transfer-reverts, or a purpose-built soulbound standard); add a superseding-reissuance flow instead of allowing revoked certificates to become valid again.

---

## 9. Institution Permissions

**Findings**

- No cascading-effect specification: when an institution is suspended, what happens to certificates already issued by its issuers? The spec implies they remain valid (§21: "revoked certificates remain on-chain... rather than being burned"), but suspension ≠ revocation, and this distinction (suspended institution's past certs stay valid unless individually revoked) should be an explicit, testable rule, not inferred.
- No rate limiting or issuance caps per institution — relevant both for gas-griefing prevention and for containing damage from a compromised institution wallet before suspension takes effect.
- No specification of **who can suspend an institution mid-dispute** vs. who can permanently reject it — `ADMIN` has both "approve" and "suspend" but no distinction between temporary administrative suspension and a for-cause permanent ban with appeal process.
- Institution metadata updates (`website`, `name`) — the storage layout allows mutation but no event (`InstitutionMetadataUpdated`) or access control note on who may update institution metadata is specified.

**Recommendation:** Add explicit suspension-cascade rules, per-institution issuance rate limits, and an `InstitutionMetadataUpdated` event with restricted access (institution self-service, admin override).

---

## 10. Smart Contract Modularity

**Findings**

- The core separation (Access / Institution / Certificate / Registry / Metadata) is sound and appropriately isolates upgrade risk — keeping `CertificateRegistry` (storage) separate from `CertificateManager` (logic) is a good pattern for minimizing storage-layout risk on logic upgrades.
- Missing modules given the findings above:
  - **RecoveryModule** — handles wallet-loss reissuance and compromised-key response (§7, §8).
  - **PauseController** — a shared, explicitly-scoped pausability contract supporting both global and per-institution pause states, rather than pause logic embedded ad hoc in each contract (§7, §9).
  - **RateLimiter** (or logic embedded in `InstitutionRegistry`) — issuance-cap enforcement per institution.
- `Treasury` and `Governance` are correctly marked "Future," but the **Timelock DAO** shown at the top of the architecture diagram is not — it's drawn as already gating `AccessController`. This is inconsistent: either the timelock is in scope for v1 (in which case it needs full specification: proposal, delay, execution, veto) or it should be marked "Future" like Treasury/Governance and v1 `SUPER_ADMIN` should instead be a defined multi-sig with no DAO layer yet.

**Recommendation:** Add `RecoveryModule` and `PauseController` as first-class contracts; resolve the Timelock DAO scope ambiguity — either fully specify it for v1 or explicitly defer it and substitute a multi-sig for launch.

---

## Revised Specification

### Revised Contract List

| Contract | Responsibility | Change from original |
|---|---|---|
| AccessController | Roles, permissions, capability-matrix grants (not strict linear inheritance) | Clarified as capability-based, not hierarchy-based |
| InstitutionRegistry | Onboarding, approval, suspension (with cascade rules), metadata, issuance rate limits | Added rate limiting, cascade rules |
| CertificateManager | Mint, revoke, reissue/supersede, verification entry point | Added `reissueCertificate()` |
| CertificateRegistry | Certificate storage, hash uniqueness, lookups | Unchanged |
| MetadataRegistry | IPFS CID, metadata URI, update events | Added `MetadataUpdated` event requirement |
| **RecoveryModule** *(new)* | Institution-attested reissuance to a new wallet on loss/compromise | New |
| **PauseController** *(new)* | Global pause + per-institution pause, single shared pausability logic | New |
| Upgrade Proxy | Transparent Proxy, admin bound to multi-sig/timelock | Governance binding clarified |
| Treasury *(future)* | Fees | Unchanged |
| Governance / Timelock DAO | Either fully specified for v1 or explicitly deferred | Scope resolved — pick one |

### Revised Role Capability Matrix

| Role | Grants | Multi-sig / Timelock required? |
|---|---|---|
| SUPER_ADMIN | Upgrade contracts, emergency global pause, assign/remove ADMIN | Yes — multi-sig + timelock mandatory |
| ADMIN | Approve/reject/suspend institutions (temporary), assign compliance holds | Recommended multi-sig for suspension of large institutions |
| INSTITUTION (identity wallet) | Manage its own ISSUER sub-accounts, update own metadata, request reissuance via RecoveryModule | No |
| ISSUER (operational sub-account) | Issue certificates within institution-set rate limits | No |
| COMPLIANCE *(new)* | Flag certificate "under review" without revoking | No |
| AUDITOR | Access revocation reasons / institution compliance history (beyond public verification) | No |
| PUBLIC | Verify certificate by ID, hash, or student wallet (read-only) | N/A |

### Revised Storage Notes

- Certificate struct: pack `revoked` (bool→bitmap), `institutionId` (uint32), `issueDate`/`expiryDate`/`createdAt` (uint40 each) into shared slots with `studentWallet`/`certificateHash` kept as full-width fields.
- Public-facing certificate identifiers should be non-sequential (hash-derived or UUID) to avoid enumeration; internal counters may remain sequential for indexing only.
- Add explicit bounds: `courseName`/`institution.name` capped (e.g., ≤128 bytes), `metadataURI` capped, `certificateHash` fixed at 32 bytes.
- Add `supersededBy` / `supersedes` fields to Certificate struct to support the reissuance flow without violating the "revoked certificates never become valid again" invariant.

### Revised Events (additions)

```text
MetadataUpdated
InstitutionMetadataUpdated
InstitutionSuspendedTemporary / InstitutionBannedPermanent   (split from single "InstitutionSuspended")
CertificateSuperseded
CertificateFlaggedUnderReview
RoleExpired
EmergencyPauseInstitution / EmergencyUnpauseInstitution      (per-institution, distinct from global Paused/Unpaused)
```

### Revised Security Controls (additions)

- Multi-sig + timelock binding for `SUPER_ADMIN` and `ProxyAdmin` — non-negotiable for v1, not deferred to "future governance."
- Initializer-lock enforced on all upgradeable implementation contracts.
- Automated storage-layout diff check as a CI gate before any upgrade is proposed to the timelock.
- Per-institution issuance rate limits, configurable by ADMIN, to contain damage from a single compromised institution key.
- Explicit external-call ordering: all `CertificateRegistry` state writes finalized before any cross-contract call to `MetadataRegistry`.

### Revised Emergency Recovery Strategy

1. **Global pause** — SUPER_ADMIN only, multi-sig, halts all state-changing operations network-wide. Reserved for systemic incidents (e.g., proxy compromise).
2. **Per-institution pause** — ADMIN or the institution itself, halts only that institution's issuance/revocation. Reserved for single-institution key compromise; does not affect other institutions or existing certificate verification.
3. **Issuer-level role revocation** — INSTITUTION can revoke a single compromised ISSUER sub-account without triggering institution-wide suspension.
4. **Student wallet recovery** — Institution-attested `reissueCertificate()` via RecoveryModule: original certificate marked `superseded` (never deleted or altered), new certificate minted to the new wallet, both linked on-chain for audit continuity. Requires institution confirmation (and optionally ADMIN co-sign for high-value credentials) to prevent unauthorized reissuance-based impersonation.

### Governance Scope Decision (must be resolved before implementation)

Pick one for v1:
- **Option A:** Fully specify Timelock DAO now — proposal creation, voting/approval threshold, execution delay, emergency veto path, and who can propose. Recommended if governance decentralization is a launch requirement.
- **Option B:** Defer DAO to a documented "Future" phase (like Treasury), and use a defined multi-sig (e.g., 3-of-5 known signers) with a fixed timelock delay for `SUPER_ADMIN`/`ProxyAdmin` actions at launch. Recommended if the priority is shipping a secure v1 quickly, with governance decentralization as a post-launch upgrade.

The current spec draws the Timelock DAO as already governing `AccessController` while treating Governance as "Future" elsewhere — this contradiction should be resolved explicitly in the next spec revision.

---

## Summary of Priority Actions

**Before implementation begins:**
1. Resolve Timelock DAO vs. multi-sig scope contradiction (Governance Scope Decision above).
2. Name the token standard for certificates explicitly.
3. Add RecoveryModule and PauseController as first-class contracts.
4. Define per-institution issuance rate limits and suspension cascade rules.
5. Specify struct packing and bounds validation explicitly in the storage layout, not as general principles.
6. Add the missing events (§5, revised list) with indexing/versioning conventions.
7. Add initializer-lock and automated storage-diff CI gate to the upgrade process.