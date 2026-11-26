# Bank OCR — User Stories

## Architecture Reference

The requirements are derived from the approved Bank OCR
architecture in `architecture/`, primarily Chapter 5
(Building Block View).

## Story Domains

| Domain | ID Prefix | Architecture Reference |
|--------|-----------|------------------------|
| Input Processing | INPUT | Chapter 5, Section 5.3.1 |
| Digit Recognition | RECOGNITION | Chapter 5, Section 5.3.2 |
| Account Validation | VALIDATION | Chapter 5, Section 5.3.3 |
| Error Correction | CORRECTION | Chapter 5, Section 5.3.4 |
| Reporting | REPORTING | Chapter 5, Section 5.3.5 |


## Story Inventory and Pareto Prioritization

The stories are derived from the Bank OCR kata and
the approved architecture in `architecture/`.

| Story ID | Title | Priority | Architecture |
|---|---|---|---|
| RECOGNITION-STORY-001 | Recognize nine-digit account numbers | Core | Chapter 5.3.2 |
| INPUT-STORY-001 | Read and parse ASCII account records | High | Chapter 5.3.1 |
| VALIDATION-STORY-001 | Validate account numbers using mod-11 | High | Chapter 5.3.3 |
| REPORTING-STORY-001 | Generate account validation reports | High | Chapter 5.3.5 |
| CORRECTION-STORY-001 | Correct invalid and illegible accounts | High | Chapter 5.3.4 |

### Pareto Analysis

- Total original stories: 5
- Core stories: 1 (20%)
- Supporting stories: 4 (80%)
- Core story: RECOGNITION-STORY-001
- Core selection criteria: central user value, MVP relevance,
  and importance for downstream validation.
- All five stories are required for complete kata coverage.

## Progress Tracking

- Core story bundles completed: 1/1 (100%)
- Total story bundles completed: 3/5 (60%)
- Completed: RECOGNITION-STORY-001 (manual), INPUT-STORY-001 (agent), VALIDATION-STORY-001 (agent)
- Remaining: REPORTING-STORY-001, CORRECTION-STORY-001
- Current phase: Part 2 — Final verification and PR preparation
