# 10. Quality Requirements

This chapter makes the quality goals from Chapter 1 testable. It follows ADR-001 and the approved Chapters 1–9. Bank OCR is a small Python 3.12 CLI kata, so every scenario below is verifiable with pytest or a simple structural check in the same codebase — no benchmark rig, production environment, or additional infrastructure is assumed.

## 10.1 Quality Requirements Overview

| ID | Quality goal | Source / binding | Meaning for Bank OCR |
|----|--------------|------------------|----------------------|
| QG-1 | Correctness / spec compliance | **Mandatory kata requirement** (K-1…K-20; Chapter 2.3) | Output matches the Coding Dojo Bank OCR specification exactly across all four user stories. Failing this is a defect, not a trade-off. |
| QG-2 | Testability | Self-imposed target (ADR-001; TC-7) | Each of the five business modules is unit-testable in isolation with pytest, favoring pure functions over hidden state. |
| QG-3 | Maintainability | Self-imposed target (ADR-001; TC-6) | Clear ownership, readable naming, no module reaching into another's internals. |
| QG-4 | Simplicity | Self-imposed target (ADR-001; TC-8, TC-9) | Simplest design that satisfies the kata: standard-library runtime, pytest for tests only, no database/broker/framework/cloud. |
| QG-5 | Determinism | Self-imposed target, with a mandatory part (TC-11; K-15) | Identical input yields identical output; `AMB` candidate lists are sorted ascending (K-15 is part of the kata). |
| QG-6 | Modularity | Self-imposed target (ADR-001; TC-2, TC-6) | Five distinct modules inside one deployable unit; not five services. |
| QG-7 | Robustness | Self-imposed target, with a mandatory vocabulary rule (Chapter 1.2; Chapter 2.4) | Structural errors are reported on stderr and do not abort the run; `ILL`/`ERR` are reserved for the kata's cases. The skip-and-continue behavior is our target, not a kata rule. |
| QG-8 | Adequate performance | Self-imposed target (Chapter 1.2; Chapter 4.1) | Single-pass processing of kata-scale input (~500 entries) without unnecessary I/O. No hard latency target exists. |

**Mandatory vs. self-imposed.** K-1…K-20 are external and non-negotiable (Chapter 2.4); QG-1 measures conformance to them. QG-2…QG-8 are our own targets derived from ADR-001 and the technical constraints. They bind this implementation but can be revised through a later ADR. Where a target overlaps the specification — sorted candidates in QG-5, status vocabulary in QG-7 — the kata part is mandatory and the remainder is ours.

## 10.2 Quality Tree

```text
Quality
├── Mandatory kata compliance
│   └── QG-1 Correctness / spec compliance (K-1…K-20)
└── Self-imposed quality targets (ADR-001)
    ├── Testability — QG-2
    ├── Design quality
    │   ├── QG-3 Maintainability
    │   └── QG-6 Modularity
    ├── QG-4 Simplicity
    ├── QG-5 Determinism (partly mandatory: K-15 sorting)
    ├── QG-7 Robustness (partly mandatory: reserved ILL/ERR vocabulary)
    └── QG-8 Adequate performance
```

The tree has no branch for security, availability, or scalability: the system is a local batch CLI with no network surface, no production users, and no service-level targets (Chapter 8.5; OC-7). Those qualities are out of scope by architecture, not silently ignored.

## 10.3 Quality Scenarios

Each scenario states a stimulus, the expected response, and measurable acceptance criteria. "Expected" always means the kata specification or a target already approved in Chapters 1–9.

| ID | Scenario (stimulus → expected response) | Measurable acceptance criteria / verification |
|----|------------------------------------------|-----------------------------------------------|
| **QG-1** | Feed the kata's published cases for all four user stories to recognition, validation, correction, and the full CLI → spec-exact digits, checksum decisions, initial classifications, and final result lines. | 100% of published kata examples produce the expected values. All ten canonical glyphs map to their digits; mod-11 cases match K-6…K-8; the K-14 example line is exactly `888888888 AMB ['888886888', '888888880', '888888988']`. No status suffix outside `AMB`/`ILL` appears in final output. Implement as pytest cases for each user story (Chapter 4.6, 8.4). |
| **QG-2** | Import each business module directly in a pytest test and call it with appropriate in-memory values, `tmp_path` files, or mocked file I/O → the module produces its documented output without running the complete CLI or starting external services. | Five pytest test modules exist, one per business module, and all pass. Input Parser tests may use pytest `tmp_path` or mocked file I/O because reading input files is its responsibility; other modules are exercised independently where applicable. No test requires a server, database, network, or shared mutable state (TC-7). |
| **QG-3** | Read or inspect each module's code and change one local rule (for example, output formatting) → the production-code change is localized to the owning module, with related unit and integration tests updated as needed. | Each module has the single responsibility documented in Chapter 5.5; no module accesses another's internals. Code inspection and the pytest suite verify behavior and current module responsibilities, but they do not prove which test files must change; reviewers confirm that production changes stay in the owning module. |
| **QG-4** | Run the full pytest suite in a clean Python 3.12 environment with only the standard library and pytest installed → tests pass with no services or packages. | Runtime dependency list contains no third-party package; no ORM, broker, web framework, or cloud SDK is imported (TC-8, TC-9; Chapter 7.5). One process, one deployment unit. |
| **QG-5** | Run the CLI twice on the same input file, and inspect an `AMB` result → identical stdout both times; candidates ascending and stable. | `stdout_from_run_1 == stdout_from_run_2` byte-for-byte. For each `AMB` case, `candidates == sorted(candidates)` and the line uses the single-quoted bracketed form of K-14/K-20. |
| **QG-6** | Inspect the import graph and deployed artifact → five separate modules wired by a thin orchestrator inside one CLI container. | A structural pytest check confirms: Input Parser is the only module reading files, Report Generator the only module formatting result lines, Error Corrector the only business module calling Recognizer/Validator, and the CLI Orchestrator contains no domain logic (Chapter 5.4). No extra module, service, or deployment unit exists. |
| **QG-7** | Provide an input file containing a short glyph line (26 characters), a non-blank fourth line, or a missing line, alongside well-formed entries → input-format error on stderr, valid entries still processed, no crash. | pytest asserts no unhandled exception; stderr contains an input-format error; the malformed entry produces no result line and no `ILL`/`ERR`/`AMB` status; every remaining well-formed entry produces its expected line (Chapters 6.4, 8.1). |
| **QG-8** | Process a generated ~500-entry input file (the kata's normal size) → one result line per well-formed record, single pass, no avoidable I/O. | The input file is opened/read once per run; the number of result lines equals the number of well-formed entries; the pytest smoke test completes without a timeout. **No fixed duration, throughput figure, or production SLA is asserted** (QG-8 has no hard target). |

**Verification notes.** All scenarios are executable in the pytest suite described in Chapters 4.6 and 8.4; none requires production data, credentials, or infrastructure. A scenario that fails QG-1 is a specification defect; a failure of QG-2…QG-8 is a quality-target regression to be weighed against ADR-001. Adding a dependency, service, or performance target to satisfy a scenario would itself require a new architectural decision (TC-5, TC-8, TC-9).
