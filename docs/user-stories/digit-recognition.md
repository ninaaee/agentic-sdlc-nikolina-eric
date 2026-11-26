Digit Recognition — Story Bundle

Original User Story

ID: RECOGNITION-STORY-001
Title: Recognize nine-digit account numbers
Domain: Digit Recognition
Priority: Core (Pareto 20%)

AS A bank operator
I WANT the system to recognize nine-digit account numbers from ASCII digit patterns
SO THAT scanned account records can be processed automatically.

Architecture references:

Chapter 5.3.2 — Digit Recognizer

Chapter 6.1 — Successful recognition and validation

Chapter 10.3 — QG-1 Correctness

SCENARIO 1: Successful account recognition

Scenario ID: RECOGNITION-STORY-001-S1

GIVEN

A well-formed record containing nine recognizable ASCII digit patterns

WHEN

The Digit Recognizer processes the nine glyphs

THEN

It returns exactly nine digits

The digits appear in their original order

The result contains no ? characters

SCENARIO 2: Preserve leading zeros

Scenario ID: RECOGNITION-STORY-001-S2

GIVEN

Nine valid ASCII glyphs representing 001234567

WHEN

The Digit Recognizer processes the record

THEN

The resulting account string is exactly "001234567"

Both leading zeros are preserved

SCENARIO 3: Unrecognizable glyph

Scenario ID: RECOGNITION-STORY-001-S3

GIVEN

A record containing eight recognizable digit glyphs

One glyph does not match any canonical digit pattern

WHEN

The Digit Recognizer processes the record

THEN

It returns a nine-character account string

The unrecognizable position contains ?

The remaining digits are unchanged and remain in order

Frontend Sub-Stories

Not applicable.

The approved Bank OCR architecture defines a command-line application without a graphical or web frontend. Input handling belongs to the Input Parser, and output formatting belongs to the Report Generator.

Reference: Chapter 5.2 — Container View; Chapter 7.4 — Deployment Interfaces.

Backend Sub-Stories

RECOGNITION-BE-001.1 — Recognize individual ASCII digits

AS A developer
I WANT each 3×3 ASCII digit pattern to be matched against the canonical digit patterns
SO THAT the system can identify digits reliably.

Architecture reference: Chapter 5.3.2 — Digit Recognizer.

SCENARIO 1: Recognize every canonical digit

Scenario ID: RECOGNITION-BE-001.1-S1

GIVEN

The ten canonical 3×3 ASCII glyph patterns for digits 0 through 9

WHEN

Each pattern is passed individually to the Digit Recognizer

THEN

Each pattern returns its corresponding digit

All ten mappings are correct
SCENARIO 2: Reject an unknown pattern

Scenario ID: RECOGNITION-BE-001.1-S2

GIVEN

A 3×3 ASCII glyph that matches none of the ten canonical patterns

WHEN

The Digit Recognizer processes the glyph

THEN

The result is ?

No digit is guessed or silently substituted

RECOGNITION-BE-001.2 — Assemble the account number

AS A developer
I WANT the nine recognized glyph results to be combined into one string
SO THAT downstream components receive a consistent account representation.

Architecture reference: Chapter 5.3.2 — Digit Recognizer; Chapter 5.4 — Pipeline Coordination.

SCENARIO 1: Assemble nine recognized digits

Scenario ID: RECOGNITION-BE-001.2-S1

GIVEN

Nine glyphs corresponding to the digits 1, 2, 3, 4, 5, 6, 7, 8, 9

WHEN

The Digit Recognizer processes the glyph sequence

THEN

The output string is exactly "123456789"

The output length is nine characters

SCENARIO 2: Preserve unknown positions

Scenario ID: RECOGNITION-BE-001.2-S2

GIVEN

Nine glyphs with one unrecognizable pattern at position three

The remaining glyphs represent 1, 2, 4, 5, 6, 7, 8, 9

WHEN

The Digit Recognizer processes the glyph sequence

THEN

The output string is exactly "12?456789"

The position of the unknown digit is preserved

Infrastructure Sub-Stories

RECOGNITION-INFRA-001.1 — Provide local recognition test fixtures

AS A developer
I WANT deterministic ASCII digit fixtures and a local test environment
SO THAT digit recognition can be verified automatically and consistently.

Architecture references:

Chapter 7.1 — Deployment Environment

Chapter 7.5 — Runtime and Development Dependencies

Chapter 10.3 — QG-1 and QG-2

SCENARIO 1: Validate canonical digit fixtures

Scenario ID: RECOGNITION-INFRA-001.1-S1

GIVEN

A local Python 3.12 environment with pytest installed

Test fixtures containing all ten canonical 3×3 ASCII digit patterns and their expected values

WHEN

The digit recognition tests are executed

THEN

Every canonical digit fixture produces its expected value

All recognition tests pass without external services

SCENARIO 2: Verify unknown-glyph fixture

Scenario ID: RECOGNITION-INFRA-001.1-S2

GIVEN

A local test fixture containing a noncanonical 3×3 ASCII glyph

WHEN

The recognition tests process the fixture

THEN

The result is ?

The test suite detects any result that differs from ?

Infrastructure scope: Local Python and pytest only. No database, network service, cloud deployment, or containerization is required.

Traceability Summary

Story or Scenario

Architecture Reference

RECOGNITION-STORY-001-S1

Chapter 5.3.2, Chapter 6.1

RECOGNITION-STORY-001-S2

Chapter 5.3.2, Chapter 8.3

RECOGNITION-STORY-001-S3

Chapter 5.3.2, Chapter 8.3

RECOGNITION-BE-001.1-S1

Chapter 5.3.2, Chapter 10.3 (QG-1)

RECOGNITION-BE-001.1-S2

Chapter 5.3.2

RECOGNITION-BE-001.2-S1

Chapter 5.3.2, Chapter 5.4

RECOGNITION-BE-001.2-S2

Chapter 5.3.2, Chapter 8.3

RECOGNITION-INFRA-001.1-S1

Chapter 7.5, Chapter 10.3 (QG-1, QG-2)

RECOGNITION-INFRA-001.1-S2

Chapter 7.5, Chapter 10.3 (QG-2)

All scenarios belong to RECOGNITION-STORY-001, directly or through their corresponding BE/INFRA sub-stories.

Scope: This bundle covers digit recognition only. Input-file parsing, checksum validation, error correction, and report formatting are addressed by their respective original user stories.
