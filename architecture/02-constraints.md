# 2. Architecture Constraints

This chapter lists the constraints under which Bank OCR is designed and built. Each constraint is classified by source so that mandatory kata requirements are kept distinct from architectural decisions we made ourselves.

- **Mandatory kata requirement (external):** Imposed by the Coding Dojo Bank OCR specification. Non-negotiable; the output must conform exactly.
- **Architectural decision (self-imposed):** Our own choice, recorded in ADR-001. Binding for this implementation, but reversible through a future ADR.
- **Organizational constraint:** Imposed by the Agentic SDLC course context.

## 2.1 Technical Constraints

These are technical constraints on the implementation. The kata specification does not mandate a language or runtime; the constraints below marked as architectural decisions are self-imposed and derive from ADR-001.

| ID | Constraint | Source |
|----|------------|--------|
| TC-1 | The system is written in **Python 3.12 only**. No other language or runtime is introduced. | Architectural decision (ADR-001) |
| TC-2 | The system is a **modular monolith**: one application, one process, one deployment unit. Responsibilities are modules, not services. | Architectural decision (ADR-001) |
| TC-3 | The system is operated through a **command-line interface (CLI)**. There is no GUI or web server in the initial implementation. | Architectural decision (ADR-001) |
| TC-4 | Input and output are **plain-text files and stdout**. No binary formats, database, ORM, cache, message broker, or cloud service is used. | Architectural decision (ADR-001) |
| TC-5 | No persistent store. Adding a database, ORM, cache, broker, or cloud infrastructure requires a new ADR that weighs trade-offs. | Architectural decision (ADR-001) |
| TC-6 | Responsibilities are kept as **distinct, testable modules**: Input Parser, Digit Recognizer, Account Validator, Error Corrector, Report Generator. Each owns one responsibility and does not reach into another's internals. | Architectural decision (ADR-001) |
| TC-7 | Modules are unit-tested in isolation with **pytest**, favoring pure functions over hidden state. | Architectural decision (ADR-001) |
| TC-8 | **Minimal dependencies.** A third-party package is added only when it removes more complexity than it introduces, and the addition is justified in an ADR. | Architectural decision (ADR-001) |
| TC-9 | No premature distribution. Microservices, serverless functions, event brokers, or cloud infrastructure are not introduced without an accepted ADR. | Architectural decision (ADR-001) |
| TC-10 | The system runs on a developer workstation under a POSIX shell (Linux). No specialized hardware is required beyond the scanner output, which arrives as text. | Organizational constraint |
| TC-11 | Execution is deterministic: identical input produces identical output, with candidate account numbers sorted for stable order. | Architectural decision (ADR-001) |

## 2.2 Organizational Constraints

These constraints come from the Agentic SDLC course setting in which Bank OCR is built.

| ID | Constraint | Source |
|----|------------|--------|
| OC-1 | **Module 2 (Part 2, Step 2) of the Agentic SDLC course delivers the software architecture documentation** for Bank OCR (this document set). The Bank OCR **implementation itself is developed in later modules**; this chapter constrains that future implementation. | Organizational constraint |
| OC-2 | A single developer (Nikolina Eric) designs, implements, and owns the system and this documentation. | Organizational constraint |
| OC-3 | The solution is evaluated against the **Coding Dojo Bank OCR kata** specification and the course criteria. Output must match the kata exactly. | Mandatory kata requirement (external) |
| OC-4 | Architecture is documented with the **arc42 template** and the **C4 model** (PlantUML diagrams rendered to SVG). | Organizational constraint |
| OC-5 | Architectural decisions are recorded as **ADRs**, following the ADR-001 structure, numbered sequentially. | Organizational constraint |
| OC-6 | Automated tests use **pytest** (per ADR-001). Continuous integration via **GitHub Actions** is planned for a later CI/CD module; it is **not** a decision recorded in ADR-001 and will be documented as its own ADR when adopted. | Organizational constraint |
| OC-7 | No production users, deployment budget, or service-level targets exist; the system is a course kata, not a production banking product. The "Bank Operator" persona (Chapter 1 stakeholders, C4 context) is illustrative, not a real user. | Organizational constraint |

## 2.3 Constraints from the Coding Dojo Bank OCR Specification

These are the **mandatory kata requirements**. They are external and non-negotiable; the implementation's output must conform exactly. The authoritative reference is the Coding Dojo Bank OCR specification: <https://codingdojo.org/kata/BankOCR/>.

They are organized by the four user stories of the specification. They are described here as constraints and are the basis for the functional goals in Chapter 1.

### 2.3.1 Input Format (User Story 1 — Parsing)

User Story 1 requires a program that takes the scanned file and parses it into actual account numbers.

| ID | Constraint |
|----|------------|
| K-1 | Each account number has **nine digits** (all in the range 0–9). |
| K-2 | Each digit is rendered as a **3×3 grid** using pipes (`\|`), underscores (`_`), and spaces. |
| K-3 | Each entry occupies **four lines**: three lines of digit glyphs followed by one **blank** line. |
| K-4 | Each non-blank line is **27 characters wide** (9 digits × 3 columns). |
| K-5 | The ten valid digits are the canonical Bank OCR glyph set (below). A glyph matching none of them is **illegible** and is represented by `?`. |

The canonical digit glyphs. Each digit is a 3×3 cell; the three rows are top, middle, and bottom. The rows are quoted so every space is explicit (this is the authoritative form):

```
Digit 0: " _ "  "| |"  "|_|"
Digit 1: "   "  "  |"  "  |"
Digit 2: " _ "  " _|"  "|_ "
Digit 3: " _ "  " _|"  " _|"
Digit 4: "   "  "|_|"  "  |"
Digit 5: " _ "  "|_ "  " _|"
Digit 6: " _ "  "|_ "  "|_|"
Digit 7: " _ "  "  |"  "  |"
Digit 8: " _ "  "|_|"  "|_|"
Digit 9: " _ "  "|_|"  " _|"
```

Rendered in sequence (digits 0–9, each 3 columns wide):

```
 _     _  _     _  _  _  _  _
| |  | _| _||_||_ |_   ||_||_|
|_|  ||_  _|  | _||_|  ||_| _|
```

> A normal input file contains around 500 entries (per the Coding Dojo spec).

### 2.3.2 Checksum Validation (User Story 2)

User Story 2 requires validating that the recognized numbers are valid account numbers using a checksum.

| ID | Constraint |
|----|------------|
| K-6 | Validity is checked with a **mod-11 checksum**. |
| K-7 | The rightmost digit is weighted by 1, the next by 2, …, up to the leftmost by 9. |
| K-8 | The account number is **valid** when the weighted sum (`d1 + 2·d2 + … + 9·d9`, with `d1` the rightmost digit) is **divisible by 11** (checksum = 0). Otherwise it is **invalid** (`ERR`). |

### 2.3.3 Output Formatting and Status Reporting (User Story 3)

User Story 3 requires writing out a file of findings, one account number per row, with a status column.

| ID | Constraint |
|----|------------|
| K-9 | **Initial classification (before any correction):** a fully recognized record with a passing checksum is *valid*; with a failing checksum it is *invalid* (`ERR`); a record containing any illegible digit (`?`) is `ILL`. Illegibility takes precedence over an invalid checksum — a `?` cannot be checksummed, so an illegible record is reported `ILL`, not `ERR`. |
| K-16 | **One result line per input record.** |
| K-17 | A valid number is printed as-is (no suffix). |
| K-18 | An invalid checksum is printed as `<number> ERR`. |
| K-19 | An illegible number is printed as `<number with ?> ILL`. |
| K-20 | An ambiguous number is printed as `<number> AMB ['c1', 'c2', …]` with candidates sorted ascending. |

The spec's User Story 3 example output:

```
457508000
664371495 ERR
86110??36 ILL
```

Illegible characters are replaced by `?`; a wrong checksum or illegible number is noted in a second column indicating status.

### 2.3.4 Single-Segment Error Correction and Ambiguity (User Story 4)

User Story 4 requires looking at numbers that came back as `ERR` or `ILL`, and trying to guess what they should be by adding or removing just one pipe or underscore. These constraints define the **final status after correction attempts**, to be read together with the initial classification in K-9.

| ID | Constraint |
|----|------------|
| K-10 | **Correction is attempted on records initially classified `ERR` or `ILL`** (User Story 4), by adding or removing exactly **one segment** (`\|` or `_`) in a single digit glyph. |
| K-11 | If exactly **one** single-segment change yields a valid account number, the **corrected number is reported with no status suffix**. |
| K-12 | If **no** single-segment change yields a valid account number, the record's final status is `ILL` — whether it was initially classified `ERR` or `ILL` (an initially illegible record keeps its `?` placeholders; an initially `ERR` record shows its readable digits with the `ILL` status). |
| K-13 | If **more than one** single-segment change yields a valid account number, the record is **ambiguous** (`AMB`). |
| K-14 | For `AMB`, all valid candidate account numbers are listed in a **sorted, single-quoted, bracketed list**, e.g. `888888888 AMB ['888886888', '888888880', '888888988']` (example from the Coding Dojo spec). |
| K-15 | Candidate account numbers are sorted in **ascending order** so output is deterministic. |

## 2.4 Distinction: Mandatory vs. Self-Imposed

The two categories must not be blurred, because they bind the project differently.

- The **Section 2.3 kata constraints (K-1 … K-20)** are mandatory and external. Changing them is not an option; they define correctness (QG-1). Any implementation choice that contradicts them is a defect, not an alternative.
- The **Section 2.1 architectural constraints (TC-1 … TC-11)** are self-imposed through ADR-001. They are binding for this implementation, but each can be revised by a subsequent ADR. For example, a future ADR could add a database (TC-5) or a web interface (TC-3) if the trade-offs justify it — but only after that ADR is accepted.
- The **Section 2.2 organizational constraints** describe the course context and documentation conventions; they govern how the work is produced and reviewed, not the runtime behavior of the system.
