# 1. Introduction and Goals

## 1.1 Overview

Bank OCR is a command-line application that reads ASCII-formatted representations of nine-digit bank account numbers produced by a scanning system, recognizes the digits, validates each account number against a checksum, attempts single-segment error correction where needed, and reports the result per record.

The application implements the Coding Dojo Bank OCR kata. It is built as part of Module 2 — Part 2, Step 2 of the Agentic SDLC course.

The architecture follows **ADR-001**: a modular monolith written in Python 3.12, exposed through a CLI, using plain-text files for input and output, with no database or distributed services, and tested with pytest. Five responsibilities are kept as distinct, testable modules — Input Parser, Digit Recognizer, Account Validator, Error Corrector, and Report Generator.

## 1.2 Requirements Overview

### Functional Goals

**FG-1 — Parse input records.**
Read a plain-text input file and split it into account records. Each account number is nine digits. Each digit is rendered as a 3×3 grid of pipes (`|`) and underscores (`_`). Each entry consists of four lines, each 27 characters wide: the first three lines represent the nine digits, and the fourth line is blank.

**FG-2 — Recognize digits.**
Map each 3×3 glyph to one of the ten digits (0–9). A glyph that matches no known digit is illegible and is represented by `?`.

**FG-3 — Validate with the checksum.**
Apply the kata's mod-11 checksum. The rightmost digit is weighted by 1, the next by 2, up to the leftmost by 9; the account number is valid when the sum of these products is divisible by 11.

**FG-4 — Classify records initially (User Story 3).**
Before any correction, each record is classified from its scan:
- A fully recognized record whose checksum passes is *valid*.
- A fully recognized record whose checksum fails is classified `ERR`.
- A record containing any `?` (illegible digit) is classified `ILL`.
Illegibility takes precedence over an invalid checksum — a `?` cannot be checksummed.

**FG-5 — Attempt single-segment correction (User Story 4).**
Records initially classified `ERR` or `ILL` are candidates for correction. Adding or removing exactly one `|` or `_` segment from a single digit glyph can repair an illegible digit or fix an invalid checksum. A correction is accepted only when it yields exactly one valid account number. The final result after correction:
- exactly one valid correction → the corrected number (no status suffix);
- more than one valid correction → ambiguous (`AMB`, see FG-6);
- no valid correction → the final status is `ILL`.

**FG-6 — Report ambiguity (User Story 4).**
When more than one single-segment correction yields a valid account number, the record is reported as ambiguous (`AMB`), listing all valid candidate account numbers as a sorted, single-quoted, bracketed list.

**FG-7 — Produce output.**
Write one result line per input record to stdout (or an output file), reflecting the final result after correction (User Story 4):
- A valid or successfully corrected number is printed as-is (no suffix).
- An ambiguous number is printed as `… AMB ['candidate1', 'candidate2', ...]` with candidates sorted in ascending order.
- A record that could not be corrected to a valid number is printed as `… ILL`; an initially illegible record keeps its `?` placeholders, while an initially `ERR` record that cannot be corrected is printed as `<number> ILL`.
The `ERR` and `ILL` labels in FG-4 are the initial classification; correction (FG-5) resolves every `ERR` or `ILL` record into a final result (corrected number, `AMB`, or `ILL`), so `ERR` does not appear in the final output.

### Quality Requirements

| ID | Quality Goal | Description |
|----|--------------|-------------|
| QG-1 | Correctness / spec compliance | Output matches the Bank OCR kata specification exactly across all four user stories, including status suffixes, ambiguity sorting, and `?` placeholders. |
| QG-2 | Testability | Each responsibility is independently unit-testable with pytest, favoring pure functions over hidden state (per ADR-001). |
| QG-3 | Maintainability | Clear separation of concerns; each module owns one responsibility and does not reach into another's internals; readable code over cleverness. |
| QG-4 | Simplicity | The simplest design that satisfies the kata. No database, ORM, cache, broker, web framework, or cloud infrastructure (per ADR-001). Adding any such dependency requires a new ADR. |
| QG-5 | Determinism | Given the same input file, the application always produces the same output. Correction and ambiguity resolution are deterministic; candidate lists are sorted so order is stable. |
| QG-6 | Modularity | Five distinct modules — Input Parser, Digit Recognizer, Account Validator, Error Corrector, Report Generator — within a single deployable unit. |
| QG-7 | Robustness | The CLI distinguishes structural input errors from kata-defined record outcomes. Malformed or truncated input files (entries that are not four lines of 27 characters, or files with missing or short lines) are detected and reported as input-format errors without unhandled crashes, and do not abort processing of the remaining records. `ILL` and `ERR` are reserved strictly for the cases defined by the Bank OCR kata: `ILL` for a record containing an illegible digit (a glyph matching no known digit) and `ERR` for a fully recognized record with a failing checksum. |
| QG-8 | Adequate performance | Processes kata-scale batch input interactively fast. No hard performance target; the solution must not introduce unnecessary I/O or processing overhead. |

## 1.3 Stakeholders

| Stakeholder | Interest in the System |
|-------------|-------------------------|
| Course participant (developer) — Nikolina Eric | Designs, implements, and owns the architecture and code; primary audience for these documents. |
| Course instructor / evaluator — Agentic SDLC course | Reviews the architecture and implementation against the Bank OCR kata and the course criteria. |
| Bank Operator (end-user persona) | Submits scanned records and reviews results through the CLI. The actor shown in the C4 system context diagram; for a kata this is a persona rather than a real user. |
| Future maintainers | Developers who later extend the system (for example, adding a web API as anticipated in ADR-001). Served by clear modularity and this documentation. |
