# Product Requirements Document (PRD)

# SkillChain
### A Decentralized Certificate Issuance & Verification Platform

Version: 1.0 (Reviewed & Improved)
Status: Draft — Post Architecture/Security Review
Blockchain: Electroneum Smart Chain (EVM Compatible)

> Reviewer's note: This revision preserves the original PRD's structure and intent while closing gaps identified in architecture, security, scalability, and UX review. New or materially changed content is marked [NEW] or [REVISED].

---

## 1. Executive Summary

SkillChain is a Web3-based digital credential platform enabling educational institutions, training organizations, professional bodies, and businesses to issue tamper-proof certificates on the Electroneum Smart Chain.

Certificate proofs are recorded on-chain; recipients retain permanent ownership via their blockchain wallets (or a managed wallet, for non-crypto-native users — [NEW]). Employers and other verifiers can validate authenticity instantly, without contacting the issuer.

SkillChain aims to reduce credential fraud, streamline verification, lower administrative costs, and establish trusted global credential infrastructure.

---

## 2. Vision Statement

To become the global standard for trusted, decentralized digital credential verification by empowering individuals with lifelong ownership of their academic and professional achievements.

---

## 3. Mission Statement

To build a secure, transparent, and scalable credential ecosystem where certificates are issued, owned, and verified instantly through blockchain technology.

---

## 4. Problem Statement

- Fake certificates are increasingly common.
- Verification is slow and expensive.
- Institutions spend significant time responding to verification requests.
- Graduates lose access to credentials if institutions discontinue services.
- Employers often cannot verify international credentials efficiently.
- Centralized databases are vulnerable to tampering and cyberattacks.
- [NEW] Non-crypto-native users are excluded from most Web3 credential systems due to wallet and gas-fee friction.

---

## 5. Proposed Solution

- Authorized organizations issue digital certificates.
- Certificates are cryptographically secured and non-transferable (soulbound) to reflect true personal ownership — [REVISED], promoted from "future feature" since it's core to the value proposition.
- Certificate ownership belongs to recipients, with a defined recovery path if wallet access is lost — [NEW].
- Employers verify credentials instantly, with no gas fee or wallet required to verify — [NEW, clarified].
- Records remain immutable at the proof level; certificate status (active/revoked/expired) is mutable via a defined state machine — [REVISED, resolves inconsistency].
- Verification requires no intermediary.

---

## 6. Business Goals

*(unchanged from v1.0 — see original)*

---

## 7. Project Objectives

*(unchanged, plus)*
- [NEW] Support institutions and students with zero prior crypto experience via gasless, embedded-wallet onboarding.

---

## 8. Stakeholders

*(unchanged from v1.0)*

---

## 9. User Personas

*(unchanged — Institution Administrator, Student, Employer, Super Admin)*

---

## 10. User Journey

Institution: Register → KYB Verification [NEW] → Wallet Connection (or managed wallet) → Create Certificate → Submit → Blockchain Transaction (async, queued) [REVISED] → Student Notified

Student: Create/Receive Wallet (self-custody or managed) → Receive Certificate → View Dashboard → Export/Share Certificate (link, QR, PDF [NEW])

Employer: Open Verification Page → Enter Certificate ID or Scan QR → View Blockchain Verification → Download Verification Report — no login or wallet required.

---

## 11. User Stories
*(original stories retained, plus)*
- [NEW] As a student, I want to recover access to my certificates if I lose my wallet, without losing ownership.
- [NEW] As an institution admin, I want to issue certificates in bulk (hundreds at once) without manual per-record submission.
- [NEW] As an employer, I want to verify a certificate without connecting a wallet or paying gas.
- [NEW] As a compliance officer, I want to request removal of personal data without breaking the integrity of on-chain proofs.

---

## 12. Functional Requirements

Authentication
- User registration and secure login
- Wallet authentication (primary for institutions/verifiers) using Sign-In with Ethereum (EIP-4361) — [NEW, resolves auth-model ambiguity]
- Optional managed/custodial wallet for students without existing wallets — [NEW]
- Role management (RBAC)
- Multi-factor authentication for administrative and institutional accounts — [NEW]

Certificate Management
- Create, issue, and batch-issue certificates — [REVISED: batch issuance is now core, not future]
- Upload and pin metadata (with defined IPFS pinning ownership — see Sec. 15) — [NEW]
- Revoke certificate (status change, not deletion) — [REVISED]
- Set expiry date, where applicable — [NEW]
- View issuance/revocation history and audit trail

Verification
- Verify by Certificate ID, QR code, or blockchain lookup
- No wallet or gas required to verify — [NEW, explicit]
- Downloadable, tamper-evident verification report (signed PDF)

Organization Management
- Institution onboarding with KYB (Know Your Business) checks — [NEW]
- Approval workflow, suspension, and appeals process — [NEW: appeals]
- Organization profiles and public issuer directory — [NEW]

Notifications
- Certificate issued / revoked / expiring soon [NEW]
- Wallet connected
- Verification completed
- Institution approval status changes — [NEW]

---

## 13. Non-Functional Requirements

*(Performance, Scalability, Availability, Reliability, Maintainability, Accessibility retained, plus explicit targets)*

- Accessibility: WCAG 2.1 AA conformance — [REVISED, made measurable]
- Availability: 99.9% uptime, with documented RTO/RPO for disaster recovery — [NEW]
- Scalability: architecture must support batch anchoring to reach millions of certificates without linear on-chain cost growth — [NEW]

---

## 14. Smart Contract Requirements — [REVISED]

- Certificate issuance and batch issuance via Merkle-root anchoring (single on-chain transaction anchors thousands of certificates; individual proofs verified off-chain against the root) — [NEW: primary scalability fix]
- Certificates implemented as non-transferable (soulbound) tokens (e.g., ERC-5192/ERC-4973 pattern) — [NEW]
- Certificate revocation via on-chain status flag, not deletion — immutability preserved at the proof layer — [REVISED, resolves core inconsistency]
- Optional expiry timestamp per certificate — [NEW]
- Role-based permissions via OpenZeppelin AccessControl (not simple Ownable) — [REVISED]
- Multisig ownership + timelock on all privileged admin functions (pause, upgrade, role changes) — [NEW: critical security fix]
- Pausable contract for emergency response — [NEW]
- Upgradeable via UUPS proxy pattern, with a documented upgrade governance process — [REVISED]
- Comprehensive event logging for all state changes
- Gas optimization audit prior to mainnet deployment
- Mandatory third-party smart contract audit + public audit report before mainnet launch — [NEW, non-negotiable acceptance gate]

---

## 15. Wallet Integration — [REVISED]

Supported:
- MetaMask, WalletConnect, Electroneum-compatible wallets
- Hardware wallet support (Ledger) for institutional admins — [NEW]
- Embedded/managed wallets (account-abstraction based, e.g., ERC-4337) for students and institutions without existing crypto wallets — [NEW: removes primary onboarding blocker]
Capabilities:
- Connect wallet, sign transactions
- Sign-In with Ethereum (EIP-4361) for authentication, with session expiry and replay protection — [NEW]
- View owned certificates
- Receive blockchain confirmations
- Wallet recovery flow: social/email-based recovery for managed wallets; guidance and re-linking process (post-KYC) for lost self-custody wallets — [NEW: closes major UX/support gap]
- Gasless transactions via meta-transaction relayer — students never need to hold native tokens to receive a certificate — [NEW]
- IPFS pinning: platform maintains pinning (via pinning service or IPFS cluster) for the life of the subscription, with a data-permanence policy disclosed to institutions — [NEW]

---

## 16. Backend Requirements

*(Authentication, business logic, API services, metadata management, organization approval, analytics, notifications, QR generation, logging, audit trails — retained)*

- [REVISED] Backend framework decision finalized at architecture sign-off (recommend NestJS for structure/testability at scale); "or Express.js" removed as an open decision before development begins.
- [NEW] Asynchronous job queue (e.g., BullMQ/Redis) for all blockchain-write operations, decoupling API response time from chain confirmation time.
- [NEW] Idempotency keys on all write endpoints to prevent duplicate issuance.

---

## 17. Frontend Requirements

*(Pages retained, plus)*
- [NEW] Multi-language support (i18n) for at least the top 3–5 target markets
- [NEW] Downloadable/shareable certificate image and PDF, styled for LinkedIn/resume use
- Responsive UI, wallet integration, dark mode, fast navigation *(retained)*

---

## 18. Database Requirements

*(Users, organizations, certificate metadata, transaction history, audit logs, notifications, roles, permissions, API keys, analytics — retained)*
- [NEW] Explicit data model separating immutable on-chain proof references from mutable, deletable off-chain personal data, to support GDPR erasure requests without breaking on-chain verification (verification checks the hash/proof, not the personal data itself).

---

## 19. API Requirements — [REVISED]

Authentication: Register, Login, Refresh Session, Wallet Sign-In (SIWE) — [NEW]

Organizations: Create, Update, Approve, Suspend, Appeal — [NEW: Appeal]

Certificates: Issue, Batch Issue — [NEW], Retrieve, Revoke, Verify

Users: Profile, Dashboard, Notifications

Analytics: Reports, Statistics, Activity logs

Cross-cutting API standards — [NEW]:
- Versioned endpoints (/v1/...)
- OpenAPI/Swagger specification published for all endpoints
- Pagination and filtering standards on all list endpoints
- Idempotency-key support on issuance/write endpoints
- Standardized error response schema
- Rate-limit headers on all responses
- Webhook support for institutions (certificate issued/revoked/verified events) — enables LMS/HR system integration

---

## 20. Admin Dashboard — [REVISED]

*(User management, institution approval, certificate analytics, revenue tracking, subscription management, system monitoring, audit logs, platform settings, support management — retained, plus)*

- [NEW] Fraud/anomaly detection alerts (unusual issuance volume, duplicate certificate patterns)
- [NEW] Institution KYB status tracker and document review queue
- [NEW] Dispute/appeals queue for revoked certificates
- [NEW] Smart contract health monitoring (pending transactions, gas price trends, relayer balance for gasless transactions)
- [NEW] Granular RBAC for internal admin team (support agent vs. compliance officer vs. super admin)
- [NEW] Feature flag management for phased rollouts
- [NEW] Exportable compliance/audit reports for regulators

---

## 21. Security Requirements

*(MFA, RBAC, wallet signature verification, HTTPS, secure API auth, rate limiting, input validation, secure file storage, smart contract audits, penetration testing, encryption, continuous monitoring — retained, plus)*
- [NEW] Multisig + timelock governance on all contract admin functions (see Sec. 14)
- [NEW] HSM or equivalent key-management service for platform signing keys and relayer wallets
- [NEW] Public bug bounty program prior to and after mainnet launch
- [NEW] Content moderation/scanning for uploaded certificate metadata and assets before IPFS pinning, to mitigate illegal-content liability
- [NEW] Documented incident response and key-compromise runbook

---

## 22. Compliance

*(GDPR, data privacy, e-signature standards, WCAG, secure SDLC — retained, plus)*
- [NEW] Explicit GDPR "right to erasure" resolution: personal data stored off-chain and deletable; on-chain records store only hashes/proofs, which are not personal data under most interpretations. This design decision should be reviewed with legal counsel prior to launch.

---

## 23. Risks

*(Technical, Business, Operational risks retained, plus)*
- [NEW] Risk: institutional adoption stalls due to crypto/gas friction — mitigated by gasless transactions and managed wallets (Sec. 15).
- [NEW] Risk: IPFS data loss if pinning lapses — mitigated by platform-managed pinning policy.

---

## 24. Assumptions

*(retained, plus)*
- [NEW] Legal counsel confirms the on-chain hash / off-chain personal data split satisfies applicable data protection law in target markets.

---

## 25. Milestones

*(Phases 1–6 retained; add explicit gate)*
- [NEW] Phase 4 (Testing) is gated on a completed, published third-party smart contract audit before proceeding to Phase 5 (Launch).

---

## 26. Future Features

*(NFT-based certificates — now promoted to core, see Sec. 14; Verifiable Credentials (W3C), DID integration, AI fraud detection, mobile apps, multi-chain support, templates, digital badges, skills portfolio, employer dashboards, API marketplace, LMS integration, cross-border recognition — retained, plus)*

- [NEW] Interoperability with existing standards (Blockcerts, Europass Digital Credentials)
- [NEW] Zero-knowledge selective disclosure (prove degree/certification without revealing underlying grades or personal detail)
- [NEW] Employer bulk-verification API for high-volume recruiters
- [NEW] Government digital-ID integration for cross-border credential recognition
- [NEW] Issuer reputation/staking system to disincentivize fraudulent institutions
- [NEW] Mobile SDK for third-party app integration

---

## 27. Success Metrics

*(retained — Business, Product, Technical KPIs, plus)*
- [NEW] Gasless transaction success rate (target ≥ 99%)
- [NEW] Median wallet-recovery resolution time
- [NEW] Zero critical/high findings unresolved from smart contract audit at launch

---

## 28. Acceptance Criteria — [REVISED, made measurable]

The product is production-ready when:
- Authorized organizations can issue certificates individually and in batch, with confirmed on-chain anchoring.
- Students receive and permanently access certificates through connected or managed wallets, including a tested recovery path.
- Employers can verify certificates instantly via ID or QR code, with no wallet or gas required, in under 3 seconds median response time.
- Smart contracts record certificate proofs on Electroneum Smart Chain and have passed a third-party security audit with no unresolved critical/high findings.
- Certificate revocation is reflected on-chain as a status change within 1 block confirmation and is immediately visible to verifiers.
- Administrative users can manage organizations, disputes, and platform settings through the admin dashboard.
- The platform meets defined security, performance (API p95 < 500ms), and reliability (99.9% uptime) targets under simulated load of [target concurrent users].
- Disaster recovery has been tested with documented RTO/RPO.
- WCAG 2.1 AA accessibility audit passed.
- GDPR data-flow design has been reviewed and signed off by legal counsel.
- User acceptance testing completed with stakeholder sign-off.
- Developer, administrator, and end-user documentation, including a published OpenAPI spec, is finalized.

---

## 29. Appendix — Technology Stack

Blockchain: Electroneum Smart Chain (EVM Compatible)
Smart Contracts: Solidity, OpenZeppelin (AccessControl, Pausable, UUPS Upgradeable)
Account Abstraction / Gasless: ERC-4337-compatible relayer — [NEW]
Frontend: React, Next.js, TypeScript, Tailwind CSS
Wallet Integration: MetaMask, WalletConnect, Electroneum-compatible wallets, embedded/managed wallet provider — [NEW]
Backend: Node.js, NestJS (finalized — [REVISED])
Database: PostgreSQL, Redis (caching + job queue)
Storage: IPFS (decentralized metadata/assets, with managed pinning), cloud object storage for application assets
Infrastructure: Docker, Nginx, GitHub Actions (CI/CD), Kubernetes (scaling), cloud hosting (AWS/Azure/GCP)
Monitoring: Prometheus, Grafana, centralized logging and alerting
Security: OpenZeppelin libraries, automated dependency scanning, periodic third-party smart contract audits, bug bounty program — [NEW]

---

## Glossary

*(retained, plus)*
- Soulbound Token: A non-transferable token bound permanently to a single wallet, used to represent credentials that cannot be sold or transferred.
- Account Abstraction (ERC-4337): A standard enabling smart-contract wallets with features like gasless transactions and social recovery.
- Merkle Root Anchoring: A batching technique where many records are hashed into a single Merkle tree, and only the root is written on-chain, drastically reducing gas cost per record.

---

## Conclusion

SkillChain is designed to modernize credential issuance and verification by combining blockchain immutability, decentralized ownership, and efficient user experience. This revision closes critical gaps around revocation semantics, wallet recovery, gasless onboarding, GDPR compatibility, and smart contract governance — positioning the platform for a secure, auditable, and genuinely scalable mainnet launch.