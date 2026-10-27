# VoidSeer

**Persistent Attack Surface Management (ASM)**  
*Discover. Remember. Correlate. Reassess.*

> **Project status:** Early design / pre-alpha. This README describes the intended architecture and roadmap, not a finished implementation.

VoidSeer is a persistent, event-driven Attack Surface Management platform written in **C**, with **SQLite** as its embedded knowledge store. It orchestrates established security tools, continuously builds an inventory of in-scope network assets, correlates observations, and schedules targeted reassessments whenever new information becomes available.

Rather than treating each scan as an isolated snapshot, VoidSeer maintains a living model of the environment: hosts, ports, services, credentials, vulnerability findings, relationships, and the evidence behind them.

## Why VoidSeer?

Traditional scanners answer *"What can I see right now?"* VoidSeer aims to answer:

- What assets and services have been observed over time?
- What changed since the last assessment?
- Which new checks became relevant because of a newly discovered host, service, credential, or vulnerability?
- Which results have already been validated, and which are stale or still uncertain?
- What should be reassessed next, without rescanning everything?

**Core idea:** New knowledge creates new, scoped assessment opportunities.

## Planned capabilities

- **Persistent network discovery:** Track in-scope hosts, exposed ports, protocols, service fingerprints, and changes across scans.
- **Tool orchestration:** Run external scanners through modular adapters and normalize their outputs into a common data model.
- **Credential-aware assessment:** Store credential *references* and metadata, associate validated access with assets and services, and schedule approved, rate-limited credential checks when applicable.
- **Vulnerability correlation:** Associate scanner findings and CVEs with affected assets, including evidence, timestamps, confidence, and validation state.
- **Incremental reassessment:** React to new findings and changed inputs by enqueueing only relevant checks.
- **History and deduplication:** Preserve observations over time, avoid repeated identical work, and distinguish current state from historical evidence.
- **Auditability:** Record what ran, against which authorized targets, with which policy, and what evidence was produced.
- **Extensible integrations:** Add new discovery, authentication-validation, and vulnerability-assessment adapters without changing the core engine.

## Tool integrations

The following integrations are **planned**, not yet guaranteed to be available:

| Tool | Intended role |
| --- | --- |
| [Nmap](https://nmap.org/) | Host discovery, port scanning, service identification |
| [Nessus](https://www.tenable.com/products/nessus) | Vulnerability scan ingestion and correlation via supported exports/APIs |
| [Hydra](https://github.com/vanhauser-thc/thc-hydra) | Strictly authorized authentication validation, under explicit policy constraints |

Adapters should treat external programs as independently installed dependencies and should not assume they are bundled or licensed with VoidSeer. Some integrations may require proprietary licenses or API access.

## How it works

```text
                 +---------------------------+
                 |  Authorized Scope/Policy  |
                 +-------------+-------------+
                               |
                               v
+----------------+   +------------------------+   +------------------+
| Tool Adapters  |-->|  Normalize & Correlate |-->| SQLite Knowledge |
| Nmap / Nessus  |   |  Findings and Evidence |   |      Base        |
| Hydra / ...    |<--|                        |<--|                  |
+--------+-------+   +------------+-----------+   +--------+---------+
         ^                        |                        |
         |                        v                        |
         |             +------------------------+          |
         +-------------| Event/Rules Engine     |<---------+
                       | Dedup / Scheduler      |
                       +------------------------+
```

Example assessment loop:

1. An authorized Nmap scan identifies a new SSH service.
2. VoidSeer records the asset and service observations in SQLite.
3. The rules engine schedules appropriate inventory and vulnerability checks.
4. An operator adds a credential reference and explicitly authorizes its validation against matching in-scope services.
5. The system records outcomes and supporting evidence; it does not automatically expand its authorization scope.
6. New Nessus findings or updated vulnerability intelligence cause potentially affected services to be reassessed.
7. Previously completed work is reused unless inputs, policy, evidence freshness, or check version have changed.

## Architecture (proposed)

| Component | Responsibility |
| --- | --- |
| Core daemon (C) | Lifecycle, configuration, orchestration, and fault handling |
| SQLite repository | Persistent assets, observations, relationships, findings, events, jobs, and audit history |
| Adapter layer | Invoke supported tools and parse structured output safely |
| Event/rules engine | Derive follow-up tasks from new or updated facts |
| Job scheduler | Prioritize work, apply concurrency limits, retries, and backoff |
| Policy engine | Enforce allowlisted scope, approved checks, time windows, and rate limits |
| CLI (planned) | Manage scope, inspections, imports, jobs, and reporting |

### Data model (initial proposal)

- `assets`: Stable asset identities and discovery metadata.
- `endpoints`: Address, protocol, port, and service observations.
- `observations`: Timestamped facts and provenance.
- `credentials`: Non-secret metadata and references to credentials held in a secure store.
- `credential_validations`: Authorized test outcomes linked to assets/endpoints.
- `vulnerabilities`: CVE identifiers, affected services, evidence, confidence, and status.
- `jobs`: Scheduled, running, completed, and failed checks.
- `events`: Newly observed facts or meaningful state changes.
- `policies`: Scope and permitted assessment actions.
- `audit_log`: Who/what initiated activity and why.

A job's deduplication identity should incorporate the target, check type, relevant inputs, tool/check version, and policy version. Findings should retain both **first seen** and **last seen** timestamps, rather than overwriting history.

## Security and responsible operation

VoidSeer is intended for **networks and systems where assessment is explicitly authorized**.

- **Deny by default:** No scanning or credential testing outside a configured allowlist.
- **Separate discovery from intrusive checks:** Authentication attempts and potentially disruptive checks require explicit opt-in policies.
- **Protect accounts:** Enforce rate limits, lockout-aware controls, concurrency caps, and stop conditions; do not attempt uncontrolled password spraying or brute forcing.
- **Protect secrets:** Prefer references to an external secret store. Do not place plaintext passwords in SQLite, logs, process arguments, or exported reports.
- **Trace actions:** Retain an audit trail, evidence provenance, and operator attribution.
- **Minimize impact:** Support maintenance windows, cancellation, and per-target resource budgets.
- **No automatic scope expansion:** Newly discovered hosts do not become authorized targets unless covered by policy.

## Technology choices

- **Language:** C (targeting C11)
- **Database:** SQLite 3 (WAL mode considered for concurrent readers)
- **Build system:** To be decided (CMake or Meson)
- **Platform:** Linux first; portability to be evaluated
- **Interfaces:** CLI first; API and dashboard are possible later milestones

## Proposed roadmap

- [ ] **M0 — Foundation:** Build skeleton, configuration, logging, SQLite migrations, and basic CLI.
- [ ] **M1 — Discovery:** Nmap adapter, XML parsing, asset/service model, and historical comparisons.
- [ ] **M2 — Persistence:** Event log, job queue, deduplication, scheduler, and recoverable execution.
- [ ] **M3 — Vulnerability intelligence:** Nessus import, CVE normalization, correlation, and finding lifecycle.
- [ ] **M4 — Credentials:** Secret-store integration, explicit authentication-validation policies, and audit records.
- [ ] **M5 — Rules:** Event-triggered reassessment based on changed assets, credentials, and vulnerability findings.
- [ ] **M6 — Reporting:** CLI views, exports, and longitudinal change reports.
- [ ] **M7 — Hardening:** Integration tests, adapter isolation, resource controls, and policy enforcement tests.

## Planned repository layout

```text
voidseer/
├── README.md
├── LICENSE                 # To be selected
├── CMakeLists.txt          # If CMake is chosen
├── include/voidseer/
├── src/
│   ├── core/
│   ├── db/
│   ├── events/
│   ├── scheduler/
│   ├── policy/
│   ├── adapters/
│   │   ├── nmap/
│   │   ├── nessus/
│   │   └── hydra/
│   └── cli/
├── migrations/
├── tests/
├── docs/
└── examples/
```

This is a **proposed** structure; no build or usage commands are published until there is a working prototype.

## Design principles

1. **Stateful by default:** Persist facts and decisions across restarts.
2. **Evidence over assumptions:** Separate observed, inferred, and validated findings.
3. **Incremental, not repetitive:** Only re-run assessments when there is a relevant reason.
4. **Adapters over reinvention:** Reuse established tools behind stable interfaces.
5. **Safety as an invariant:** Every task is checked against authorization and impact policy.
6. **Explainable decisions:** Be able to answer *why* any job was scheduled.

## Contributing

VoidSeer is currently a project concept. Feedback on the event model, SQLite schema, adapter interfaces, and scheduling strategy is welcome once the repository is opened for contributions.

## License

**TBD.** A license will be selected before the first public release.

---

**VoidSeer** — *The void sees what the network forgets.*
