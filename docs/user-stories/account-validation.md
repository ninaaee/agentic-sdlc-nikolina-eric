# Account Validation — Story Bundle

## Original User Story

**ID**: VALIDATION-STORY-001
**Title**: Validate account numbers using mod-11
**Domain**: Account Validation
**Priority**: High (supporting)
**Parent Story**: N/A (original story)
**Architecture References**: Chapter 5.3.3 Account Validator; Chapter 2.3.2 Checksum Validation (User Story 2); Chapter 8.3 Data Representation and Validation; Chapter 10.3 QG-1 Correctness

AS A bank operator
I WANT the system to apply the mod-11 checksum to each recognized nine-digit account number
SO THAT valid account numbers are distinguished from invalid ones before any correction is attempted.

**Checksum rule (Chapter 2.3.2, K-6…K-8).** For a nine-character string, weight the rightmost digit by 1, the next by 2, …, up to the leftmost digit by 9; sum the nine products. The account number is **valid** when that sum is divisible by 11 (checksum result 0). Otherwise the initial classification is `ERR`. A string containing `?` is classified `ILL` and is **not** checksummed, because illegibility takes precedence (Chapter 5.3.3; K-9).

### SCENARIO 1: Valid nine-digit account number

**Scenario ID**: VALIDATION-STORY-001-S1
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 5.3.3 Account Validator; Chapter 2.3.2 K-6…K-8

**GIVEN**
* The recognized account string `457508000` (the kata's valid example)
* The string contains nine digits and no `?`

**WHEN**
* The Account Validator applies the mod-11 checksum

**THEN**
* The weighted sum is 187, computed with left-to-right weights 9, 8, 7, 6, 5, 4, 3, 2, 1
* 187 is divisible by 11, so the checksum result is 0
* The initial classification is `valid`
* No `ERR` or `ILL` classification is returned

### SCENARIO 2: Invalid nine-digit account number

**Scenario ID**: VALIDATION-STORY-001-S2
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 5.3.3 Account Validator; Chapter 2.3.2 K-8; Chapter 6.2 Scenario 2

**GIVEN**
* The recognized account string `664371495` (the kata's invalid example)
* The string contains nine legible digits and no `?`

**WHEN**
* The Account Validator applies the mod-11 checksum

**THEN**
* The weighted sum is 222
* 222 is not divisible by 11 (remainder 2), so the checksum result is 2
* The initial classification is `ERR`
* The classification is not `ILL`, because no digit is illegible

### SCENARIO 3: Account string containing an illegible digit

**Scenario ID**: VALIDATION-STORY-001-S3
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 5.3.3 Account Validator; Chapter 2.3.3 K-9; Chapter 8.3 Data Representation and Validation; Chapter 6.2 Scenario 2

**GIVEN**
* The recognized account string `86110??36` (the kata's illegible example)
* At least one position contains `?`

**WHEN**
* The Account Validator classifies the string

**THEN**
* The initial classification is `ILL`
* No mod-11 checksum result is computed or used for that string
* The classification is `ILL` rather than `ERR`, even if the legible digits would fail the checksum (illegibility precedence)

## Frontend Sub-Stories

Not applicable for the approved Bank OCR CLI. Validation is a pure function called in-process by the CLI Orchestrator; it has no user-facing control, screen, or browser component (Chapter 5.2 Container View; Chapter 12 Glossary, "CLI Orchestrator" and "Account Validator"). The only external interface is a local file plus stdout/stderr (Chapter 7.4 Deployment Interfaces), and a GUI or REST/RPC API is out of scope (Chapter 3.3.2; TC-3). No FE story ID is created.

## Backend Sub-Stories

### VALIDATION-BE-001.1 — Compute the weighted mod-11 checksum

**ID**: VALIDATION-BE-001.1
**Parent Story**: VALIDATION-STORY-001
**Architecture References**: Chapter 5.3.3 Account Validator; Chapter 2.3.2 K-6…K-8; Chapter 8.3 Data Representation and Validation

AS A developer
I WANT an exact weighted-sum checksum over a nine-digit string
SO THAT validity is decided by the kata's formula rather than by an approximation.

### SCENARIO 1: Sum uses right-to-left weights 1 through 9

**Scenario ID**: VALIDATION-BE-001.1-S1
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 2.3.2 K-7, K-8; Chapter 5.3.3; Chapter 10.3 QG-1

**GIVEN**
* The in-memory string `457508000`

**WHEN**
* The checksum function multiplies each digit by its weight (rightmost = 1, leftmost = 9) and sums the products

**THEN**
* The returned sum equals 187
* The returned checksum remainder is 0
* The function accepts a nine-character string and returns plain numeric values, with no hidden state (Chapter 8.3)

### SCENARIO 2: Non-zero remainder for a failing checksum

**Scenario ID**: VALIDATION-BE-001.1-S2
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 2.3.2 K-8; Chapter 5.3.3

**GIVEN**
* The in-memory string `664371495`

**WHEN**
* The checksum function is called with the same weight order

**THEN**
* The returned sum equals 222
* The returned checksum remainder is 2
* The function reports the number as not passing the checksum

### VALIDATION-BE-001.2 — Assign the initial classification

**ID**: VALIDATION-BE-001.2
**Parent Story**: VALIDATION-STORY-001
**Architecture References**: Chapter 5.3.3 Account Validator; Chapter 1.2 FG-4; Chapter 6.2 Scenario 2; Chapter 2.3.3 K-9

AS A developer
I WANT the checksum result and the presence of `?` mapped to one of `valid`, `ERR`, or `ILL`
SO THAT the pipeline has a single, testable initial classification per record.

### SCENARIO 1: Passing checksum classifies as valid

**Scenario ID**: VALIDATION-BE-001.2-S1
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 5.3.3; Chapter 1.2 FG-4; Chapter 10.3 QG-1

**GIVEN**
* A fully legible nine-digit string whose checksum remainder is 0

**WHEN**
* The Account Validator classifies the string

**THEN**
* The result is `valid`
* The result is not `ERR` or `ILL`

### SCENARIO 2: Failing checksum classifies as ERR

**Scenario ID**: VALIDATION-BE-001.2-S2
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 5.3.3; Chapter 2.3.2 K-8; Chapter 1.2 FG-4

**GIVEN**
* A fully legible nine-digit string whose checksum remainder is not 0

**WHEN**
* The Account Validator classifies the string

**THEN**
* The result is the initial classification `ERR`
* The result is an internal classification, not a final output line

### SCENARIO 3: Illegible string classifies as ILL without checksumming

**Scenario ID**: VALIDATION-BE-001.2-S3
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 5.3.3; Chapter 2.3.3 K-9; Chapter 8.3

**GIVEN**
* A nine-character string containing at least one `?`

**WHEN**
* The Account Validator classifies the string

**THEN**
* The result is `ILL`
* No weighted sum is used to decide the classification
* The result is not `ERR`

### VALIDATION-BE-001.3 — Keep validation independent of correction

**ID**: VALIDATION-BE-001.3
**Parent Story**: VALIDATION-STORY-001
**Architecture References**: Chapter 5.3.3 Account Validator (Boundary); Chapter 5.3.4 Error Corrector; Chapter 6.2 Scenario 2; Chapter 10.3 QG-6

AS A developer
I WANT the validator to return only the initial classification and checksum result
SO THAT correction remains a separate, independently testable responsibility.

### SCENARIO 1: No correction behaviour in the validator result

**Scenario ID**: VALIDATION-BE-001.3-S1
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 5.3.3; Chapter 5.3.4; Chapter 6.2; Chapter 10.3 QG-6

**GIVEN**
* The failing string `664371495`, which receives the initial classification `ERR`

**WHEN**
* The Account Validator is called directly from a pytest test

**THEN**
* The returned value contains only the classification and checksum result
* It contains no corrected account number, no `AMB` candidate list, and no final `ILL` outcome
* The validator does not call the Digit Recognizer or the Error Corrector (Chapter 5.3.3 "Dependencies")

## Infrastructure Sub-Stories

All four infrastructure categories were considered. One relevant, testable sub-story is included; the other three are explicitly not applicable for this pure-function domain.

### VALIDATION-INFRA-001.1 — Local pytest environment for checksum verification

**ID**: VALIDATION-INFRA-001.1
**Parent Story**: VALIDATION-STORY-001
**Infrastructure Category**: Deployment
**Architecture References**: Chapter 7.1 Deployment Environment; Chapter 7.5 Runtime and Development Dependencies; Chapter 9 ADR-001; Chapter 10.3 QG-1 and QG-2

AS A developer
I WANT to run validator tests locally with Python 3.12 and pytest
SO THAT the kata's checksum examples are verified without any external service.

### SCENARIO 1: Verify the kata's published validation examples

**Scenario ID**: VALIDATION-INFRA-001.1-S1
**Parent Story**: VALIDATION-STORY-001
**Architecture Reference**: Chapter 7.1; Chapter 7.5; Chapter 10.3 QG-1, QG-2; Chapter 2.3.2

**GIVEN**
* A local Python 3.12 environment with pytest installed
* In-memory strings `457508000`, `664371495`, and `86110??36` from the kata examples

**WHEN**
* The validator test module is executed

**THEN**
* The test asserts `valid`, `ERR`, and `ILL` respectively for those three strings
* All validator tests pass with no server, database, network, file I/O, or container required (ADR-001; TC-2, TC-7)

### Not applicable: Data storage

The Account Validator operates only on an in-memory nine-character string and returns plain numeric/classification values; it reads no file and persists no state (Chapter 8.3 Data Representation and Validation; Chapter 2.1 TC-4 and TC-5). File-backed fixtures belong to `INPUT-STORY-001`; no data-store sub-story is created here.

### Not applicable: Events

Checksum validation has no file or stream trigger of its own; it is invoked by a plain in-process call from the CLI Orchestrator, with no broker, queue, webhook, or asynchronous messaging (Chapter 5.4 Pipeline Coordination by the CLI; Chapter 6.5 Cross-scenario runtime conventions). Call order is covered by the pipeline stories, so no event sub-story is created here.

### Not applicable: Monitoring

The validator emits no stdout or stderr text: result-line formatting belongs to the Report Generator and structural input-format errors to the CLI Orchestrator (Chapter 8.2 Logging and Diagnostics; Chapter 5.3.5 Report Generator; Chapter 6.4 Scenario 4). No logging framework or diagnostic store is introduced (Chapter 8.2), so no monitoring sub-story is created here.

## Traceability Summary

| Item ID | Parent Story | Architecture Reference | Verifiable Expected Result |
|---------|--------------|------------------------|----------------------------|
| VALIDATION-STORY-001-S1 | VALIDATION-STORY-001 | Chapter 5.3.3; Chapter 2.3.2 K-6…K-8 | `457508000` sums to 187, remainder 0, classified `valid` |
| VALIDATION-STORY-001-S2 | VALIDATION-STORY-001 | Chapter 5.3.3; Chapter 2.3.2 K-8; Chapter 6.2 | `664371495` sums to 222, remainder 2, classified `ERR` |
| VALIDATION-STORY-001-S3 | VALIDATION-STORY-001 | Chapter 5.3.3; Chapter 2.3.3 K-9; Chapter 8.3 | `86110??36` classified `ILL` with no checksum applied |
| VALIDATION-BE-001.1-S1 | VALIDATION-STORY-001 | Chapter 2.3.2 K-7, K-8; Chapter 10.3 QG-1 | Sum is 187 with rightmost weight 1 and leftmost weight 9 |
| VALIDATION-BE-001.1-S2 | VALIDATION-STORY-001 | Chapter 2.3.2 K-8; Chapter 5.3.3 | Sum is 222, remainder 2, reported as not passing |
| VALIDATION-BE-001.2-S1 | VALIDATION-STORY-001 | Chapter 5.3.3; Chapter 1.2 FG-4 | Zero remainder yields `valid`, not `ERR`/`ILL` |
| VALIDATION-BE-001.2-S2 | VALIDATION-STORY-001 | Chapter 5.3.3; Chapter 2.3.2 K-8 | Non-zero remainder yields initial classification `ERR` |
| VALIDATION-BE-001.2-S3 | VALIDATION-STORY-001 | Chapter 5.3.3; Chapter 2.3.3 K-9; Chapter 8.3 | String with `?` yields `ILL` and skips checksumming |
| VALIDATION-BE-001.3-S1 | VALIDATION-STORY-001 | Chapter 5.3.3; Chapter 5.3.4; Chapter 10.3 QG-6 | Validator returns no corrected number, candidates, or final status and calls no other module |
| VALIDATION-INFRA-001.1-S1 | VALIDATION-STORY-001 | Chapter 7.1; Chapter 7.5; Chapter 10.3 QG-1, QG-2 | Local pytest asserts `valid`/`ERR`/`ILL` for the three kata examples with no services |

Traceability chain used throughout: `architecture section → VALIDATION-STORY-001 → VALIDATION-BE/VALIDATION-INFRA sub-story → scenario ID → verifiable THEN assertion`.

## Scope

This bundle covers the mod-11 weighted checksum, the initial classification (`valid` / `ERR` / `ILL`), illegibility precedence, and local verification of those rules. `ERR` here is always an **initial** classification; resolving it by single-segment changes belongs to `CORRECTION-STORY-001` (Chapter 5.3.4). Glyph-to-digit mapping and `?` origin belong to `RECOGNITION-STORY-001` (Chapter 5.3.2); file reading and structural validation belong to `INPUT-STORY-001` (Chapter 5.3.1); formatting final result lines and status suffixes belongs to `REPORTING-STORY-001` (Chapter 5.3.5). Databases, APIs, brokers, GUIs, and cloud deployment are outside the boundary (Chapter 3.3.2; ADR-001 in Chapter 9) and are not introduced here.
