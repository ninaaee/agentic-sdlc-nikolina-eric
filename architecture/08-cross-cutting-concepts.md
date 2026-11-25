# 8. Cross-cutting Concepts

This chapter describes the concepts that apply across all modules of Bank OCR rather than to a single module. Everything here follows **ADR-001** and the approved Chapters 1–7: one Python 3.12 modular monolith, one CLI process, plain-text I/O, five business modules, and a thin orchestrator. No new module, service, framework, or infrastructure is introduced.

## 8.1 Error Handling and Robustness

Bank OCR keeps two fundamentally different kinds of "error" separate (QG-7; Chapters 1.2, 4.5, 6.4):

| Category | Cause | Detected by | Reported as | Effect on the run |
|----------|-------|-------------|-------------|-------------------|
| Structural input error | A file entry is not three 27-character glyph lines plus a blank line, or a line is missing/short (QG-7; K-3, K-4) | Input Parser | Short input-format error message on **stderr** (Chapter 6.4, 7.4) | The malformed entry is skipped; processing continues with the next entry. No unhandled exception. |
| Kata-defined record outcome | A glyph matches no canonical digit, a checksum fails, or several corrections are valid | Digit Recognizer → Account Validator → Error Corrector | A final result line on **stdout**: corrected number, `… AMB [...]`, or `… ILL` (FG-4…FG-7; K-9…K-20) | Normal record processing; one result line per well-formed record (K-16). |

The distinction matters because `ILL`, `ERR`, and `AMB` are vocabulary defined by the kata, not general failure labels:

- `ILL` is the initial classification when a record contains `?`, and also the final result when no single-segment correction yields a valid number (FG-4, FG-5; K-9, K-12).
- `ERR` is only an initial classification: a fully recognized record whose checksum fails. It is resolved by correction and **never appears in final output** (FG-7; Chapter 6.2).
- `AMB` is a final result when more than one single-segment correction is valid (FG-6; K-13, K-14).

**Runtime convention.** A structural problem is reported directly by the thin CLI Orchestrator on stderr; it is not passed to the Digit Recognizer, Account Validator, Error Corrector, or Report Generator as if it were a record (Chapter 6.4). This keeps structural errors distinct from kata statuses and keeps stdout reserved for result lines (Chapter 7.4).

**Robustness without machinery.** There is no exception framework, retry policy, or error-handling module. The parser validates structure at the boundary, the orchestrator reports the problem, and the loop moves on — the simplest behavior that satisfies QG-7. An uncorrectable *kata* record is not an application failure; it produces the kata's `ILL` result.

## 8.2 Logging and Diagnostics

Diagnostics in Bank OCR are deliberately minimal:

- **stdout** carries only final result lines — one per well-formed input record (K-16; Chapter 7.4). It is the machine-readable contract with the kata.
- **stderr** carries input-format error messages for structurally malformed entries (Chapters 6.4, 7.4). The operator can see that an entry was skipped without polluting the result stream.
- No diagnostic text is interleaved with result lines, and no status suffix is invented for structural errors.

No logging framework is introduced. A third-party logging package would add a runtime dependency contrary to minimal dependencies (TC-8), and a separate logging module would duplicate the thin orchestrator's existing role of reporting a short stderr message (ADR-001; Chapter 5.4). Plain stderr output from the CLI Orchestrator is sufficient for a local, single-process batch tool with no long-running service to monitor (OC-7).

There is no audit log, telemetry, or persistent diagnostic store (TC-4, TC-5). Deterministic output is preserved: identical input produces identical stdout (QG-5; TC-11).

## 8.3 Data Representation and Validation

Modules exchange small, plain Python values — no custom stateful objects, shared caches, or hidden state (TC-7; Chapter 6.5):

| Concept | Representation | Owner | Validation / rule |
|---------|----------------|-------|-------------------|
| Input record | Nine 3×3 glyph cells, each three strings of three characters from `{&#124;`, `_`, space}; entry is three glyph rows plus a blank line | Input Parser | Exactly 27 characters per glyph row, four lines per entry (K-2…K-4); structural violations become input-format errors |
| Digit / account number | Nine-character string; digits `0`…`9`, or `?` for a glyph matching none of the ten canonical patterns | Digit Recognizer | Exact match against the canonical glyph set (K-5, FG-2); no probabilistic recognition (Chapter 4.4) |
| Initial classification | One of *valid*, `ERR`, `ILL` | Account Validator | Mod-11 checksum: rightmost digit weighted 1 … leftmost weighted 9; divisible by 11 means valid (K-6…K-8). `?` takes precedence: a record containing `?` is `ILL` and is not checksummed (K-9) |
| Correction outcome | A corrected nine-digit number, an `AMB` result carrying all valid candidates, or `ILL` | Error Corrector | Exactly one valid single-segment change → number; more than one → `AMB`; none → `ILL` (K-10…K-13) |
| Ambiguous candidates | Sorted, single-quoted, bracketed list of candidate account numbers | Error Corrector → Report Generator | Ascending sort before the result leaves the corrector (K-14, K-15); the formatter does not re-sort (Chapter 6.3) |
| Result line | One text line per well-formed record: number, `… AMB ['…', '…']`, or `… ILL` | Report Generator | Formats only the final result; no recognition, validation, or correction (Chapter 5.3.5) |

The same nine-character account string flows through recognition, validation, and correction; the same nine-glyph-cell record flows from the parser to the recognizer and, for `ERR`/`ILL` records, to the Error Corrector (Chapter 5.5). Because modules are pure functions over these values, no module reaches into another's internals and results depend only on inputs (QG-3, QG-6; TC-6, TC-7).

Determinism is enforced by design: exact glyph matching, the fixed mod-11 formula, exhaustive enumeration of single-segment changes, and ascending candidate sorting leave no room for randomness or ordering differences (QG-5; Chapter 4.4). No ORM, schema, or serialized state exists — data representation is confined to the running process and the kata's text files (TC-4, TC-5).

## 8.4 Testing

ADR-001 fixes pytest as the test tool, and TC-7 requires each responsibility to be unit-testable in isolation, favoring pure functions over hidden state. The test approach has two layers:

**Unit tests (one area per module).**

| Module under test | Typical assertions |
|-------------------|--------------------|
| Input Parser | Correct splitting of four-line entries; detection of short lines, missing blank lines, and over-wide glyph rows. |
| Digit Recognizer | Each of the ten canonical glyphs maps to its digit; an unmatched glyph maps to `?`. |
| Account Validator | Mod-11 acceptance and rejection; `?` classified `ILL` even when other digits exist (illegibility precedence). |
| Error Corrector | Adding/removing one segment finds exactly one candidate, several candidates, or none; candidates are sorted ascending for `AMB`. |
| Report Generator | Exact line formatting for number, `AMB`, and `ILL`, including `?` placeholders. |

Tests supply inputs directly and assert outputs; they do not set up databases, servers, or shared fixtures with hidden state (TC-7, QG-2).

**Integration / specification tests.** End-to-end tests run the CLI pipeline against the kata's published sample files and compare the produced result lines with the specification's expected output: parsing and recognition (User Story 1), checksum validation and initial status reporting (User Stories 2–3), and single-segment correction with ambiguity (User Story 4) (Chapter 4.6; Chapter 2.3). A robustness case supplies malformed input and checks that an input-format error is reported on stderr while the remaining valid records still produce their result lines (QG-7; Chapter 6.4). These tests verify the contract at the file/stream boundary described in Chapter 7.4.

There is no separate test framework, test server, or fixture database. pytest covers both layers within the same Python 3.12 codebase (ADR-001; TC-1, TC-8). Continuous integration via GitHub Actions is planned for a later course module and is not part of this concept or ADR-001 (OC-6).

## 8.5 Security

Bank OCR has no authentication or authorization requirement (ADR-001; TC-3; OC-7):

- It is a **local CLI** invoked by one operator on a workstation. There is no web server, API, network listener, or remote client, so there is no external identity to authenticate.
- There are no user accounts, roles, permissions, or protected resources to authorize against, and no persistent data store to secure (TC-5).
- The Bank Operator is a kata persona, not a production user; there are no production users or service-level obligations (OC-7).

Adding a login system, token handling, or an identity provider would introduce a new external surface, dependencies, and an ADR-level decision with no benefit for the kata (TC-8, TC-9). Security is therefore out of scope by architecture, not by omission.

**Safe handling of input files without infrastructure.** The operator supplies the input path on the command line, and the scanner supplies the file's content (Chapter 7.4). Practical handling stays at the file boundary:

- The CLI reads the given file as **text data**; it does not execute, import, or interpret its contents, and it does not invoke a shell on them.
- The Input Parser validates the four-line/27-character structure before any glyph is passed onward; unexpected content becomes a `?` at recognition or an input-format error at parsing, never code execution (Chapters 5.3.1, 8.1).
- The input file is treated as read-only by Bank OCR; result text is emitted to stdout or to the operator-selected output target, and nothing is written back to the input (TC-4).
- No secrets, credentials, or configuration files are required; no network fetch or filesystem-wide scanning occurs (TC-3, TC-5).

These are disciplined file-handling conventions, not a security subsystem: no sandbox, container, encryption, or access-control layer is justified for a local batch kata (TC-9).

## 8.6 Maintainability

Maintainability follows the separation already approved in Chapter 5:

| Concern | Where it lives | Rule |
|---------|----------------|------|
| CLI invocation, wiring, stderr diagnostics | CLI Orchestrator | Thin entry point; no domain logic; holds no state between records (Chapter 5.4). |
| File reading and structural validation | Input Parser | Only module that reads the input file; owns record boundaries (Chapter 5.3.1). |
| Domain logic: recognition, checksum/classification, correction | Digit Recognizer, Account Validator, Error Corrector | Pure functions over plain values; Error Corrector reuses the other two, and no other module-to-module calls exist (Chapters 5.3.2–5.3.4). |
| Output formatting | Report Generator | Only module that formats result lines; no recognition, validation, or correction (Chapter 5.3.5). |

This keeps domain logic separate from file I/O and CLI orchestration: a change to glyph matching touches only the Digit Recognizer, a change to output formatting only the Report Generator (QG-3, QG-6). Clear names and one responsibility per module are preferred over cleverness (ADR-001; QG-3).

Dependencies stay minimal: the Python 3.12 standard library at runtime, pytest for tests only, and no third-party runtime package without an ADR (TC-1, TC-8; Chapter 7.5). One language, one process, one deployment unit means no cross-language boundaries, service contracts, or distributed debugging (TC-2, TC-9). Tests document each module's behavior, so the same small set of concepts is visible in code and in documentation (QG-2).

## 8.7 Concept Summary

| Concept | Practical rule | Source |
|---------|----------------|--------|
| Error handling | Structural errors → stderr and skip the entry; `ILL`/`ERR`/`AMB` only for kata record outcomes; `ERR` never in final output. | QG-7; Chapters 4.5, 6.1–6.4 |
| Diagnostics | stdout = result lines only; stderr = input-format errors; no logging framework. | Chapters 6.5, 7.4 |
| Data | Nine 3×3 glyph cells → nine-character account string → initial classification → correction outcome → result line; mod-11 and sorted candidates are deterministic. | Chapters 2.3, 5.5, 6.3 |
| Testing | pytest unit tests per module plus end-to-end tests against the kata's published cases. | ADR-001; TC-7; Chapter 4.6 |
| Security | No authn/authz: local CLI, no network, no accounts; read text files as data and validate their structure. | TC-3, TC-9; OC-7; Chapter 7.6 |
| Maintainability | Five focused modules plus a thin orchestrator; pure functions; standard library runtime and pytest for tests only. | ADR-001; TC-2, TC-6, TC-8; Chapter 5 |

No database, API, cloud service, authentication system, or additional business module is introduced by any concept in this chapter.
