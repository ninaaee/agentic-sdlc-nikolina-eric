# 11. Risks and Technical Debts

This chapter records risks and technical debt for the approved Bank OCR architecture. It is derived from **ADR-001** and Chapters 1–10 and introduces no new architectural decision.

Two categories are kept separate:

- **Potential risks** are things that may go wrong while implementing or evolving the approved design.
- **Actual technical debt** is known deferred work or a simplification we have deliberately accepted and would repay if the project continues.

No production-scale, cloud, security, or distributed-system problems are listed: the system is a local Python 3.12 CLI kata with one process, plain-text I/O, no network surface, and no production users (ADR-001; OC-7; TC-3–TC-5, TC-9).

## 11.1 Potential Risks

| ID | Description | Likelihood | Impact | Mitigation strategy | Relevant components / goals |
|----|-------------|------------|--------|---------------------|-----------------------------|
| **R-1** | **Specification mismatch.** The original kata has exact glyph patterns, mod-11 weighting, status vocabulary, and result-line formatting; a small misunderstanding can produce plausible but incorrect output. | Medium | High — this fails QG-1, the mandatory kata contract. | Maintain pytest cases for all four user stories and the exact published examples, including the K-14 `AMB` line. Check changes against the K-1…K-20 requirements in Chapter 1.2 before accepting them. | Digit Recognizer, Account Validator, Report Generator; QG-1, QG-5 |
| **R-2** | **Error-correction complexity.** Single-segment correction must enumerate changes, re-recognize glyphs, re-check checksums, and distinguish one valid candidate, several, or none. Ordering or duplicate handling can corrupt `AMB` output. | Medium | High — wrong corrected numbers or unstable ambiguity results violate the kata. | Keep the Error Corrector pure; unit-test one-candidate, many-candidate, and no-candidate cases; reuse the Recognizer and Validator as in Chapter 5.5; assert ascending sorted candidates from K-15. Do not add unspecified heuristics. | Error Corrector, Digit Recognizer, Account Validator; QG-1, QG-2, QG-5 |
| **R-3** | **Ambiguous and illegible status confusion.** `ILL` is both an initial `?` classification and a final no-correction result; `ERR` is only initial and never final; `AMB` is only for multiple valid corrections. Structural input errors are not kata statuses. | Medium | High for spec output; Medium for operator diagnostics. | Keep the status taxonomy from Chapter 8.1 in tests: `?` precedence for `ILL`, no final `ERR`, exact `AMB` formatting, and stderr-only input-format errors. | Recognizer, Validator, Corrector, Report Generator, CLI Orchestrator; QG-1, QG-7 |
| **R-4** | **Malformed input parsing.** Short or over-wide glyph rows, missing blank lines, or a missing final line can misalign records and silently distort recognition. | Medium | Medium — skipped entries are expected, misparsed entries are not. | Validate the four-line, 27-character-per-row boundary in the Input Parser; unit-test with pytest `tmp_path` or mocked file I/O; skip malformed entries, report them on stderr, and continue (QG-7). | Input Parser, CLI Orchestrator; QG-2, QG-7 |
| **R-5** | **Integration drift between the five modules.** The hand-offs between glyph cells, nine-character account strings, initial classifications, correction outcomes, and result lines are simple but must stay consistent. | Low–Medium | Medium — module tests can pass while the CLI pipeline fails. | Keep the thin orchestrator and the call graph in Chapter 5.5; add full-CLI integration tests using the kata samples; add contract tests for each hand-off. | CLI Orchestrator and all five business modules; QG-2, QG-3, QG-6 |
| **R-6** | **Test-coverage and maintainability drift.** New rules may be added without pytest coverage, or domain logic may leak into the orchestrator/report generator as the code grows. | Medium | Medium — regressions become harder to locate. | Keep one pytest area per business module plus specification-level tests; review changes against the single-responsibility table in Chapter 5.5; keep runtime dependencies to the standard library (TC-8). | All modules; QG-2, QG-3, QG-4, QG-6 |
| **R-7** | **Limitations of the Python CLI architecture.** The application expects a local file hand-off and command-line invocation; it has no API, GUI, persistent state, or automated external trigger. | Low | Low–Medium if users expect richer integration. | Keep the scope explicit in Chapters 3, 7, and 8; use stdout redirection or the optional output target; treat any new interface as a new ADR-level decision rather than an implicit extension. | CLI Orchestrator, Report Generator; QG-4, TC-3–TC-5, TC-9, OC-7 |
| **R-8** | **Input growth beyond kata scale.** A much larger file, or repeated reading of the same file, could make the simple batch design inefficient. | Low | Low for this kata; Medium only if scope changes. | Preserve single-pass processing and one read per run; keep the ~500-entry smoke scenario from Chapter 10; revisit only with concrete evidence of larger inputs. | Input Parser, CLI Orchestrator; QG-8, QG-4 |

These risks are manageable within the approved design. None requires a database, service split, framework, cloud resource, or authentication system.

## 11.2 Actual Technical Debt

Only one known item qualifies as actual technical debt under the accepted architecture. It is a deferred implementation activity, not a change to ADR-001.

| ID | Description | Likelihood / status | Impact | Mitigation strategy | Relevant components / goals |
|----|-------------|---------------------|--------|---------------------|-----------------------------|
| **TD-1** | **Automated CI not yet wired.** pytest is the required test tool and Chapter 10 defines verifiable scenarios, but GitHub Actions is deferred to a later course module and is not part of ADR-001. Until then, the suite must be run manually. | Present today by accepted deferral (OC-6). | Low–Medium — regressions can be merged unnoticed if local runs are skipped. | Run the full pytest suite locally before accepting changes. Add automated test execution when the later CI/CD module calls for it; if CI is formally adopted as architecture, record that decision separately. | pytest suite, all five business modules; QG-2, QG-3, QG-4; OC-6 |

No other code-level debt is asserted by this architecture document: the modules, interfaces, and dependency rules are still at the design level, and no implementation shortcut has been accepted.

## 11.3 Intentional Scope, Not Technical Debt

The following are deliberate ADR-001 constraints, not debts to repay:

- No database, ORM, cache, broker, web framework, API, or cloud service (TC-5, TC-8, TC-9).
- No authentication, authorization, or production-user model for a local CLI (Chapter 8.5; OC-7).
- No logging framework: stdout is reserved for result lines and stderr for input-format errors (Chapter 8.2).
- No microservices, serverless functions, containers, or production deployment pipeline (ADR-001; Chapter 7).
- No fixed latency, throughput, or availability guarantee for kata-scale input (QG-8; Chapter 10).

## 11.4 Review Triggers

Revisit this chapter if the kata specification changes, tests reveal a specification mismatch, correction rules expand, input structure changes, CI is adopted, or a new external interface is requested. Otherwise the mitigations above remain sufficient for the local Bank OCR CLI.

**Scope statement.** This chapter adds no new module, service, dependency, infrastructure, or architectural decision. It only records realistic risks and the single accepted deferred item.
