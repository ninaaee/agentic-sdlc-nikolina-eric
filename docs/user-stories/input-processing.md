# Input Processing — Story Bundle

## Original User Story

**ID**: INPUT-STORY-001
**Title**: Read and parse ASCII account records
**Domain**: Input Processing
**Priority**: High (supporting)
**Parent Story**: N/A (original story)
**Architecture References**: Chapter 5.3.1 Input Parser; Chapter 6.4 Scenario 4 — Handling malformed input without crashing the application; Chapter 8.1 Error Handling and Robustness; Chapter 2.3.1 Input Format (User Story 1 — Parsing); Chapter 10.3 QG-7 Robustness

AS A bank operator
I WANT the system to read a plain-text scan file and split it into four-line ASCII account records
SO THAT each scanned account number can be passed to the recognition pipeline as a set of nine glyph cells.

### SCENARIO 1: Parse one well-formed ASCII account record

**Scenario ID**: INPUT-STORY-001-S1
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 5.3.1 Input Parser; Chapter 2.3.1 K-1…K-4; Chapter 1.2 FG-1

**GIVEN**
* A local ASCII text file containing one entry of three 27-character glyph lines followed by one blank line (K-3, K-4)
* The three glyph lines contain the nine 3×3 digit patterns for a valid nine-digit account

**WHEN**
* The Input Parser reads that file and splits its contents into records

**THEN**
* Exactly one record is returned
* The record contains exactly nine 3×3 glyph cells, in left-to-right order
* Each glyph cell is three strings of three characters drawn from `|`, `_`, and space (Chapter 8.3)
* No structural-error marker is produced for the entry

### SCENARIO 2: Split multiple records in file order

**Scenario ID**: INPUT-STORY-001-S2
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 5.3.1 Input Parser; Chapter 5.4 Pipeline Coordination by the CLI; Chapter 7.3 Deployable Unit and Startup; Chapter 10.3 QG-8

**GIVEN**
* A local ASCII text file containing two well-formed four-line entries, one after the other

**WHEN**
* The Input Parser reads the whole file during a single CLI invocation

**THEN**
* The parser returns a sequence of exactly two records
* The two records appear in the same order as in the input file
* Each returned record contains nine 3×3 glyph cells
* Neither entry is merged with, dropped from, or reordered relative to the other

### SCENARIO 3: Report a malformed entry and continue processing

**Scenario ID**: INPUT-STORY-001-S3
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 6.4 Scenario 4 — Handling malformed input without crashing the application; Chapter 8.1 Error Handling and Robustness; Chapter 10.3 QG-7

**GIVEN**
* A local ASCII text file containing one structurally malformed entry (a glyph line of 26 characters) followed by one well-formed entry

**WHEN**
* The Input Parser reads the file and the CLI Orchestrator processes the entries in order

**THEN**
* The malformed entry produces a structural-error marker and no glyph cells
* The CLI Orchestrator writes an input-format error to stderr
* The well-formed entry after it is still returned as a record with nine glyph cells
* No unhandled exception is raised and the run is not aborted (QG-7)
* No `ILL`, `ERR`, or `AMB` result line is emitted for the malformed entry

## Frontend Sub-Stories

Not applicable for the approved Bank OCR CLI. There is no graphical, browser-facing, or web component in this architecture: the only container is the Python 3.12 Bank OCR CLI (Chapter 5.2 Container View), its sole input interface is a local ASCII text file passed by path (Chapter 7.4 Deployment Interfaces), and the supplied input is read and split by the Input Parser rather than by any UI control (Chapter 12 Glossary, "Input Parser" and "ASCII account record"). A GUI or REST/RPC API is explicitly outside the system boundary (Chapter 3.3.2 Outside the boundary; TC-3 in Chapter 2.1). No frontend story ID is created; input handling is covered entirely by the backend and infrastructure sub-stories below.

## Backend Sub-Stories

### INPUT-BE-001.1 — Validate the four-line glyph entry structure

**ID**: INPUT-BE-001.1
**Parent Story**: INPUT-STORY-001
**Architecture References**: Chapter 5.3.1 Input Parser; Chapter 2.3.1 K-3 and K-4; Chapter 8.3 Data Representation and Validation

AS A developer
I WANT each entry's three 27-character glyph lines and blank fourth line to be checked at the parsing boundary
SO THAT only structurally valid records reach the Digit Recognizer.

### SCENARIO 1: Accept a structurally valid entry

**Scenario ID**: INPUT-BE-001.1-S1
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 5.3.1 Input Parser; Chapter 2.3.1 K-3, K-4; Chapter 8.3

**GIVEN**
* An in-memory four-line entry whose first three lines are exactly 27 characters and whose fourth line is empty

**WHEN**
* The Input Parser validates the entry

**THEN**
* The entry is accepted as one record
* The record yields nine glyph cells of three rows by three columns
* The cells preserve the original glyph row order and column positions (K-2…K-4)

### SCENARIO 2: Reject a short glyph line

**Scenario ID**: INPUT-BE-001.1-S2
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 5.3.1 Input Parser; Chapter 8.1 Error Handling and Robustness; Chapter 10.3 QG-7

**GIVEN**
* A four-line entry whose first glyph line has 26 characters instead of 27

**WHEN**
* The Input Parser validates the entry

**THEN**
* The entry is not emitted as a record
* A structural-error marker is returned for that entry
* No glyph cells from the short line are passed to the Digit Recognizer

### SCENARIO 3: Reject a non-blank fourth line

**Scenario ID**: INPUT-BE-001.1-S3
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 5.3.1 Input Parser; Chapter 2.3.1 K-3; Chapter 10.3 QG-7

**GIVEN**
* An entry whose first three lines are 27 characters wide but whose fourth line contains a non-blank character

**WHEN**
* The Input Parser validates the entry

**THEN**
* The entry is not emitted as a record
* A structural-error marker is returned for that entry
* The fourth line is not interpreted as a glyph row

### INPUT-BE-001.2 — Read the input file and emit records in order

**ID**: INPUT-BE-001.2
**Parent Story**: INPUT-STORY-001
**Architecture References**: Chapter 5.3.1 Input Parser; Chapter 5.4 Pipeline Coordination by the CLI; Chapter 7.4 Deployment Interfaces (Files and Streams); Chapter 10.3 QG-8

AS A developer
I WANT the parser to read the supplied text file once and return its records in file order
SO THAT downstream modules receive a deterministic, single-pass record sequence.

### SCENARIO 1: Return records in file order

**Scenario ID**: INPUT-BE-001.2-S1
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 5.3.1 Input Parser; Chapter 5.4 Pipeline Coordination by the CLI; Chapter 8.3

**GIVEN**
* An input file path pointing to three consecutive well-formed four-line entries

**WHEN**
* The Input Parser reads the file and returns its record sequence to the CLI Orchestrator

**THEN**
* The returned sequence has length three
* Record *i* in the sequence corresponds to entry *i* in the file
* Each record contains nine glyph cells and no data from a neighbouring entry

### SCENARIO 2: Read the file once in a single pass

**Scenario ID**: INPUT-BE-001.2-S2
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 7.3 Deployable Unit and Startup; Chapter 10.3 QG-8; Chapter 8.4 Testing

**GIVEN**
* A mocked file reader supplied to the Input Parser with a multi-entry text file

**WHEN**
* The parser produces the full record sequence

**THEN**
* The input file is opened and read exactly once
* The number of returned records equals the number of well-formed entries
* No second read pass or repeated I/O of the input file is observed (QG-8)

### INPUT-BE-001.3 — Keep structural errors distinct from kata outcomes

**ID**: INPUT-BE-001.3
**Parent Story**: INPUT-STORY-001
**Architecture References**: Chapter 8.1 Error Handling and Robustness; Chapter 6.4 Scenario 4 — Handling malformed input without crashing the application; Chapter 12 Glossary ("stderr", "Final result"); Chapter 10.3 QG-7

AS A developer
I WANT structural input errors to be represented as parser markers rather than as record classifications
SO THAT `ILL`, `ERR`, and `AMB` remain reserved for kata-defined outcomes only.

### SCENARIO 1: Marker is not a kata status

**Scenario ID**: INPUT-BE-001.3-S1
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 8.1 Error Handling and Robustness; Chapter 6.4 Scenario 4; Chapter 10.3 QG-7

**GIVEN**
* An entry with a missing blank fourth line

**WHEN**
* The Input Parser processes the entry and the CLI Orchestrator receives its result

**THEN**
* The result is a structural-error marker, not an account-number string
* The marker does not equal `ILL`, `ERR`, or `AMB`
* The Report Generator receives no result line for that entry
* An input-format error is available to the orchestrator for reporting on stderr

## Infrastructure Sub-Stories

All four infrastructure categories were considered for this story, and each applies: parsing runs as a local pytest-tested module, reads a local text file, is invoked in-process by the CLI Orchestrator, and surfaces structural errors on stderr. No category is omitted, and no category is replaced with a fictional service.

### INPUT-INFRA-001.1 — Local Python 3.12 pytest environment for parser tests

**ID**: INPUT-INFRA-001.1
**Parent Story**: INPUT-STORY-001
**Infrastructure Category**: Deployment
**Architecture References**: Chapter 7.1 Deployment Environment; Chapter 7.5 Runtime and Development Dependencies; Chapter 9 ADR-001; Chapter 10.3 QG-2

AS A developer
I WANT a local Python 3.12 workstation with pytest to run Input Parser unit tests
SO THAT parsing behavior is verified without any deployment unit beyond the CLI.

### SCENARIO 1: Run parser tests locally with no services

**Scenario ID**: INPUT-INFRA-001.1-S1
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 7.1; Chapter 7.5; Chapter 9 ADR-001; Chapter 10.3 QG-2

**GIVEN**
* A local Python 3.12 environment with pytest installed and no running service, container, or database

**WHEN**
* The Input Parser unit tests are executed from a POSIX shell

**THEN**
* The parser test module passes
* No server, container image, cloud resource, or network port is required for the tests to pass (ADR-001; TC-2, TC-9)

### INPUT-INFRA-001.2 — Local text-file fixtures without a persistent store

**ID**: INPUT-INFRA-001.2
**Parent Story**: INPUT-STORY-001
**Infrastructure Category**: Data storage
**Architecture References**: Chapter 7.4 Deployment Interfaces (Files and Streams); Chapter 8.3 Data Representation and Validation; Chapter 2.1 TC-4 and TC-5; Chapter 10.3 QG-2

AS A developer
I WANT to create ASCII account-record fixtures with pytest `tmp_path`
SO THAT file reading is tested against real local text without introducing a database.

### SCENARIO 1: Parse a record from a temporary text file

**Scenario ID**: INPUT-INFRA-001.2-S1
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 7.4; Chapter 8.3; Chapter 2.3.1 K-1…K-4; Chapter 10.3 QG-2

**GIVEN**
* A pytest `tmp_path` file containing one well-formed four-line ASCII account record

**WHEN**
* The test passes that file path to the Input Parser

**THEN**
* The returned in-memory record contains nine glyph cells
* The asserted cell values equal the glyphs written to the file
* No database, ORM, cache, broker, or persistent application state is created (TC-4, TC-5)

### SCENARIO 2: Input file remains unchanged after parsing

**Scenario ID**: INPUT-INFRA-001.2-S2
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 8.5 Security (safe handling of input files); Chapter 2.1 TC-4; Chapter 7.4

**GIVEN**
* A pytest `tmp_path` file with one well-formed record and a captured copy of its exact bytes

**WHEN**
* The test runs the Input Parser against that file path

**THEN**
* The file's bytes are identical before and after the parse
* The parsing result exists only as in-memory values in the running test
* No output is written back into the input file

### INPUT-INFRA-001.3 — In-process parser invocation with no event broker

**ID**: INPUT-INFRA-001.3
**Parent Story**: INPUT-STORY-001
**Infrastructure Category**: Events
**Architecture References**: Chapter 5.4 Pipeline Coordination by the CLI; Chapter 6.5 Cross-scenario runtime conventions; Chapter 7.4 Deployment Interfaces (Files and Streams); Chapter 10.3 QG-6

AS A developer
I WANT the parser to be invoked by a single in-process call from the CLI Orchestrator
SO THAT record flow is sequential and testable without a broker or asynchronous messaging.

### SCENARIO 1: Verify the parser is the sole file reader in the pipeline

**Scenario ID**: INPUT-INFRA-001.3-S1
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 5.4; Chapter 6.5; Chapter 10.3 QG-6; Chapter 8.6 Maintainability

**GIVEN**
* A pytest structural check over the five business modules and the CLI Orchestrator

**WHEN**
* The check inspects module file-reading calls following one CLI invocation with an input file path

**THEN**
* The Input Parser is the only business module that reads the input file
* The parser is invoked as a plain Python function call returning an in-memory sequence (Chapter 6.5)
* No queue, event broker, webhook, IPC, network call, or asynchronous message is used

### INPUT-INFRA-001.4 — stdout/stderr separation for input-format errors

**ID**: INPUT-INFRA-001.4
**Parent Story**: INPUT-STORY-001
**Infrastructure Category**: Monitoring
**Architecture References**: Chapter 8.2 Logging and Diagnostics; Chapter 6.4 Scenario 4 — Handling malformed input without crashing the application; Chapter 7.4 Deployment Interfaces (Files and Streams); Chapter 10.3 QG-7

AS A developer
I WANT structural input errors on stderr and result lines on stdout
SO THAT a skipped entry is visible without polluting the kata result stream.

### SCENARIO 1: Capture the two output streams during a CLI run

**Scenario ID**: INPUT-INFRA-001.4-S1
**Parent Story**: INPUT-STORY-001
**Architecture Reference**: Chapter 8.2; Chapter 6.4; Chapter 7.4; Chapter 10.3 QG-7

**GIVEN**
* A local input file with one short-line entry followed by one well-formed valid entry
* A pytest test that captures stdout and stderr with `capsys`

**WHEN**
* The CLI processes the file

**THEN**
* stderr contains an input-format error message for the malformed entry
* stdout contains only the result line for the well-formed entry
* stdout contains no `ILL`, `ERR`, or `AMB` line for the malformed entry
* No logging framework, telemetry, or persistent diagnostic store is required (Chapter 8.2)

## Traceability Summary

| Item ID | Parent Story | Architecture Reference | Verifiable Expected Result |
|---------|--------------|------------------------|----------------------------|
| INPUT-STORY-001-S1 | INPUT-STORY-001 | Chapter 5.3.1; Chapter 2.3.1 K-1…K-4 | One record with nine ordered 3×3 glyph cells and no error marker |
| INPUT-STORY-001-S2 | INPUT-STORY-001 | Chapter 5.3.1; Chapter 5.4; Chapter 10.3 QG-8 | Two records returned in file order, each with nine glyph cells |
| INPUT-STORY-001-S3 | INPUT-STORY-001 | Chapter 6.4; Chapter 8.1; Chapter 10.3 QG-7 | Error on stderr, later valid record still parsed, no crash, no kata status line |
| INPUT-BE-001.1-S1 | INPUT-STORY-001 | Chapter 5.3.1; Chapter 2.3.1 K-3, K-4 | Valid four-line entry yields nine ordered glyph cells |
| INPUT-BE-001.1-S2 | INPUT-STORY-001 | Chapter 5.3.1; Chapter 10.3 QG-7 | 26-character line yields a structural-error marker, not a record |
| INPUT-BE-001.1-S3 | INPUT-STORY-001 | Chapter 5.3.1; Chapter 2.3.1 K-3 | Non-blank fourth line yields a structural-error marker |
| INPUT-BE-001.2-S1 | INPUT-STORY-001 | Chapter 5.3.1; Chapter 5.4; Chapter 8.3 | Three entries produce three records in file order |
| INPUT-BE-001.2-S2 | INPUT-STORY-001 | Chapter 7.3; Chapter 10.3 QG-8 | Input file opened/read once; record count equals well-formed entry count |
| INPUT-BE-001.3-S1 | INPUT-STORY-001 | Chapter 8.1; Chapter 6.4; Chapter 10.3 QG-7 | Marker is not `ILL`/`ERR`/`AMB` and produces no result line |
| INPUT-INFRA-001.1-S1 | INPUT-STORY-001 | Chapter 7.1; Chapter 7.5; Chapter 9 ADR-001 | Parser pytest module passes with no service, container, or database |
| INPUT-INFRA-001.2-S1 | INPUT-STORY-001 | Chapter 7.4; Chapter 8.3; Chapter 10.3 QG-2 | `tmp_path` file parses to nine asserted glyph cells with no persistent store |
| INPUT-INFRA-001.2-S2 | INPUT-STORY-001 | Chapter 8.5; Chapter 2.1 TC-4; Chapter 7.4 | Input file bytes unchanged after parsing; results stay in memory |
| INPUT-INFRA-001.3-S1 | INPUT-STORY-001 | Chapter 5.4; Chapter 6.5; Chapter 10.3 QG-6 | Structural check confirms parser is sole file reader; in-process call, no broker |
| INPUT-INFRA-001.4-S1 | INPUT-STORY-001 | Chapter 8.2; Chapter 6.4; Chapter 10.3 QG-7 | `capsys` shows error on stderr and only the valid result line on stdout |

Traceability chain used throughout: `architecture section → INPUT-STORY-001 → INPUT-BE/INPUT-INFRA sub-story → scenario ID → verifiable THEN assertion`.

## Scope

This bundle covers input-file reading, four-line/27-character structural validation, multiple-record splitting, structural-error marking, and the local test environment and stdout/stderr boundary for parsing only. Digit recognition (glyph-to-digit mapping and `?` for illegible glyphs) belongs to `RECOGNITION-STORY-001` (Chapter 5.3.2). Mod-11 checksum and initial classification belong to `VALIDATION-STORY-001` (Chapter 5.3.3). Single-segment correction and ambiguity belong to `CORRECTION-STORY-001` (Chapter 5.3.4). Result-line formatting and status suffixes belong to `REPORTING-STORY-001` (Chapter 5.3.5). Physical scanning, databases, web/GUI interfaces, event brokers, and cloud deployment are outside the boundary (Chapter 3.3.2; ADR-001 in Chapter 9) and are not introduced here.
