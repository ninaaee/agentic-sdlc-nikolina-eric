# 12. Glossary

This glossary defines the architectural and Bank OCR terms used in the documentation. Definitions are based on **ADR-001** and approved Chapters 1–11 and introduce no new requirements, decisions, or implementation details. Terms are ordered alphabetically.

| Term | Definition |
|------|------------|
| **Account number** | The nine-digit bank account identifier processed by Bank OCR. After recognition it is represented as a nine-character string; an unrecognized digit is represented by `?` until correction is attempted. |
| **Account Validator** | Business module that applies the mod-11 checksum and assigns the initial classification: valid, `ERR`, or `ILL`. A record containing `?` is `ILL` and is not checksummed. |
| **ADR** | Architecture Decision Record: a short record of one architectural decision. **ADR-001**, "Use a Modular Python CLI Architecture for Bank OCR," is the accepted source of truth for this architecture. |
| **AMB** | "Ambiguous": the final result when more than one valid single-segment correction exists. The result carries all valid candidates, sorted ascending, in the kata's bracketed output form. |
| **arc42** | The documentation template used for this architecture. This document set follows its chapter structure, including quality requirements, risks, and glossary. |
| **ASCII account record** | One input entry representing one account number: three 27-character glyph lines plus a blank line. The Input Parser splits these records from the plain-text input file. |
| **ASCII glyph** | A 3×3 pattern made of vertical bars, underscores, and spaces that depicts one digit. Exact matches map to `0`–`9`; an unmatched glyph maps to `?`. |
| **Bank OCR** | The Bank OCR System: a Python 3.12 command-line application that reads ASCII-formatted account records, recognizes digits, validates checksums, corrects recoverable errors, and reports one result line per well-formed record. |
| **Business module** | One of the five responsibility-specific modules inside the CLI: Input Parser, Digit Recognizer, Account Validator, Error Corrector, and Report Generator. They are modules in one application, not services, and are distinct from the CLI Orchestrator. |
| **C4 model** | A model for describing software architecture at nested levels: System Context, Container, Component, and Deployment views. The Bank OCR diagrams use C4-PlantUML and match the approved chapters. |
| **Checksum / mod-11** | Validation rule for a nine-digit account number: weight the rightmost digit by 1 through the leftmost by 9, sum the products, and accept the number when the sum is divisible by 11. It is applied by the Account Validator. |
| **CLI Orchestrator** | The thin entry point that wires and drives the five business modules, calls them in the approved order, and reports structural input errors on stderr. It contains no domain logic and is not one of the five business modules. |
| **Component (C4)** | A building block inside a C4 Container. In Bank OCR, the C4 components are the CLI Orchestrator and the five business modules; they are Python modules, not separately deployed services. |
| **Container (C4)** | A deployable or runnable unit in the C4 model. Bank OCR has exactly one C4 Container: the Python 3.12 Bank OCR CLI. This term does **not** mean a Docker container. |
| **Correction candidate** | A nine-character account number produced by one valid single-segment change that passes the checksum. Multiple candidates produce an `AMB` result with candidates sorted ascending. |
| **Digit Recognizer** | Business module that maps each 3×3 ASCII glyph to a digit by exact match, or to `?` when no canonical pattern matches. |
| **Docker container (not used)** | A containerization runtime unrelated to the C4 term **Container**. Docker is not used by Bank OCR; the sole deployment unit is the Python 3.12 CLI application on a developer workstation. |
| **ERR** | An **initial classification** meaning "checksum error": the record is fully legible but fails mod-11 validation. `ERR` is resolved by the Error Corrector and never appears in final output. |
| **Error Corrector** | Business module that attempts single-segment corrections for `ERR` and `ILL` records. It reuses the Digit Recognizer and Account Validator to evaluate candidates and returns a corrected number, `AMB`, or final `ILL`. |
| **Final result** | The result line emitted after correction: a valid account number, an `AMB` result with sorted candidates, or `ILL`. `ERR` is not a final result, and structural input errors are reported on stderr instead of as result lines. |
| **ILL** | "Illegible." As an initial classification, it means the recognized account contains `?` and is not checksummed. As a final status after correction, it means zero valid correction candidates were found; exactly one valid candidate yields a corrected account number, and more than one valid candidate yields `AMB`. |
| **Illegible digit** | A glyph that does not match any canonical digit pattern and is represented as `?`. This is distinct from an **invalid account number**, where all digits are legible but the checksum fails. |
| **Initial classification** | The Account Validator's pre-correction result: valid, `ERR`, or `ILL`. It is internal pipeline state, not the same as the final result produced after correction. |
| **Input Parser** | Business module that reads the input file, validates record structure, and splits it into ASCII account records. It is the only business module responsible for reading input files. |
| **Input record** | See **ASCII account record**: three glyph lines plus a blank line, representing one account number. |
| **Invalid account number** | A fully recognized nine-digit account number that fails the mod-11 checksum. It is initially classified `ERR`, unlike an account containing an illegible digit, which is classified `ILL`. |
| **Modular monolith** | The chosen architecture: one application, one process, one deployment unit, with responsibilities separated into modules rather than services. Bank OCR is a Python 3.12 CLI modular monolith. |
| **PlantUML** | The text-based diagramming tool used to author `.puml` sources under `architecture/diagrams/` and render them to `.svg` files. The diagrams use the C4-PlantUML library. |
| **Report Generator** | Business module that formats the final result line for a record. It performs no recognition, validation, or correction and does not decide kata statuses. |
| **Single-segment correction** | Adding or removing one segment in one 3×3 glyph to form a candidate account number, which is then re-recognized and checksum-checked by the Error Corrector. |
| **stderr** | The standard error stream. Bank OCR uses it for input-format error messages about structurally malformed entries; these messages are not `ERR`, `ILL`, or `AMB` result statuses. |
| **stdout** | The standard output stream. Bank OCR writes final result lines here, one per well-formed record; it is kept separate from diagnostics. |

No term in this glossary changes ADR-001 or the approved Chapters 1–11.
