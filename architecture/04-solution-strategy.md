# 4. Solution Strategy

This chapter summarizes the overarching architectural approach for Bank OCR and explains how it addresses the quality goals defined in Chapter 1. The individual decisions are recorded in ADR-001 and operate within the constraints of Chapter 2; the runtime shape matches the C4 context diagram in Chapter 3. The chapter stays at the strategy level — building-block detail follows in Chapter 5.

The guiding principle (ADR-001) is to choose the simplest design that satisfies the kata: one language, one process, one deployment unit, plain-text I/O, and a small set of well-separated modules — adding nothing ahead of need.

## 4.1 Modular Monolith in Python 3.12

Bank OCR is a single Python 3.12 application, run as one process and deployed as one unit (ADR-001; TC-1, TC-2). Responsibilities are modules within that unit, not separate services.

**Rationale.** The kata is local, batch-oriented text processing with no concurrency, scale, or distribution requirements. A monolith keeps build, run, and deployment trivial, and Python's built-in string and file handling fits the task directly. Splitting the system into services, serverless functions, or cloud components would add complexity the kata does not justify (TC-9).

**Quality goals addressed.** QG-4 (simplicity) and QG-3 (maintainability): one language and one unit minimize moving parts. It also keeps performance adequate for kata-scale input (QG-8): a single pass over around 500 entries (per the spec) is interactively fast, with no infrastructure overhead.

## 4.2 Command-Line Interface with Plain-Text File I/O

Interaction is through a CLI: the operator points Bank OCR at an input file and reads results from stdout (or an output file) (ADR-001; TC-3, TC-4). This is exactly the shape of the C4 context diagram in Chapter 3 — the Bank Operator drives the CLI, and the Existing Scanning System supplies the ASCII input file.

**Rationale.** The kata defines its input and output as text. A CLI with file I/O matches that directly and needs no database, ORM, cache, or broker (TC-5). No persistent state is required; state lives only for the duration of a run.

**Quality goals addressed.** QG-4 (simplicity — no persistence layer or framework) and QG-1 (spec compliance — the input and output formats are the kata's own).

## 4.3 Separation into Five Independently Testable Modules

The monolith is organized into five modules, each owning one responsibility and not reaching into another's internals (ADR-001; TC-6):

1. **Input Parser** — splits the input file into account records.
2. **Digit Recognizer** — maps 3×3 glyphs to digits (or `?`).
3. **Account Validator** — applies the checksum and classifies records.
4. **Error Corrector** — attempts single-segment correction and resolves ambiguity.
5. **Report Generator** — formats the final results.

Modules favor pure functions over hidden state (TC-7). The component-level relationships are detailed in Chapter 5.

**Rationale.** One responsibility per module makes each unit-testable in isolation and keeps changes localized. Pure functions make behavior reproducible from their inputs, which is the foundation of deterministic testing.

**Quality goals addressed.** QG-6 (modularity), QG-2 (testability), and QG-3 (maintainability — clear ownership and naming over cleverness).

## 4.4 Deterministic Recognition, Validation, and Correction

The three core computations are deterministic by construction, so identical input always yields identical output (QG-5; TC-11):

- **Digit recognition** uses exact matching against the ten canonical 3×3 glyphs (K-5); a glyph matching none is `?`. There is no probabilistic or learned model.
- **Checksum validation** applies the kata's mod-11 formula directly (K-6…K-8).
- **Error correction** enumerates every single-segment change (add or remove one `|` or `_`) to each digit of an `ERR`/`ILL` record, keeps the candidates that yield a valid account number, and applies the kata's outcome rules: exactly one → corrected number; more than one → `AMB`; none → `ILL` (K-10…K-15). When several candidates exist, they are sorted in ascending order so the `AMB` list is stable.

The pipeline applies these in sequence: initial classification (User Story 3: valid / `ERR` / `ILL`, FG-4) is followed by correction (User Story 4, FG-5/FG-6), and the final output reflects the result after correction (FG-7).

**Quality goals addressed.** QG-1 (spec compliance — the rules are the kata's, applied exactly) and QG-5 (determinism — no randomness, no hidden state, sorted ambiguity).

## 4.5 Error Handling Strategy

Bank OCR keeps two categories of failure separate (QG-7):

- **Structural input errors** — malformed or truncated files (entries whose first three lines are not exactly 27 characters each or whose fourth line is not blank, or files with missing or short lines). These are reported as input-format errors, do not crash the run, and do not abort the remaining records.
- **Kata record outcomes** — `ILL`, `ERR`, and `AMB`. These are reserved strictly for the cases the kata defines (QG-7): `ILL`/`ERR` are the initial classification (FG-4), and `AMB`/`ILL` are the final results after correction (FG-5, FG-6).

The initial classification (User Story 3) is distinguished from the final result after correction (User Story 4): an initially `ERR` or `ILL` record that cannot be corrected is reported `ILL` in the final output (FG-7).

**Quality goals addressed.** QG-7 (robustness — structural errors isolated from record outcomes, no unhandled crashes) and QG-1 (correctness — statuses used only where the kata defines them).

## 4.6 Testing Strategy

Each module is unit-tested in isolation with pytest, favoring pure functions so tests supply inputs and assert outputs without setting up shared state (ADR-001; TC-7). The kata's published test cases (use case 1 for parsing and recognition; use cases 3 and 4 for correction and ambiguity) serve as the spec-compliance checks (QG-1). Continuous integration via GitHub Actions is planned for a later CI/CD module and is not part of ADR-001 (OC-6).

**Quality goals addressed.** QG-2 (testability — isolated module tests), QG-1 (spec compliance — kata test cases), and QG-3 (maintainability — tests document each responsibility).

## 4.7 How the Strategy Meets the Quality Goals

| Quality goal (Chapter 1) | How the solution strategy addresses it |
|--------------------------|----------------------------------------|
| QG-1 Correctness / spec compliance | Exact glyph matching, mod-11 validation, exhaustive single-segment correction (§4.4); I/O in the kata's text formats (§4.2); kata test cases as acceptance checks (§4.6). |
| QG-2 Testability | Five independent modules, pure functions, isolated pytest unit tests (§4.3, §4.6). |
| QG-3 Maintainability | One responsibility per module, clear naming, single language, tests that document behavior (§4.1, §4.3, §4.6). |
| QG-4 Simplicity | Modular monolith, CLI, plain-text I/O, no database/framework/cloud, minimal dependencies (§4.1, §4.2; TC-8). |
| QG-5 Determinism | Deterministic recognition/validation/correction; sorted ambiguity candidates (§4.4; TC-11). |
| QG-6 Modularity | Five distinct responsibilities as modules in one deployable unit (§4.3). |
| QG-7 Robustness | Structural input errors separated from kata outcomes; remaining records still processed (§4.5). |
| QG-8 Adequate performance | Single-pass batch processing over kata-scale input (~500 entries) with no unnecessary I/O (§4.1). |
