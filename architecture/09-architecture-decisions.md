# 9. Architecture Decisions

## ADR-001: Use a Modular Python CLI Architecture for Bank OCR

**Status:** Accepted

### Context

The Bank OCR kata requires processing ASCII-formatted representations of nine-digit bank account numbers.

The application must recognize digits, validate account numbers using a checksum, detect invalid or illegible records, and attempt error correction.

The processing workflow consists of several logically separate responsibilities, but does not require distributed services, persistent database storage, or network communication.

The solution should remain simple, maintainable, and easy to test.

### Decision

We will implement Bank OCR as a modular monolithic application using Python 3.12.

The application will expose a command-line interface (CLI) and use plain text files for input and output.

The architecture will separate the following responsibilities:

- **Input Parser:** Reads and separates ASCII account records.
- **Digit Recognizer:** Converts ASCII digit patterns into numeric characters.
- **Account Validator:** Checks account numbers using the required checksum algorithm.
- **Error Corrector:** Attempts to correct invalid or illegible digits.
- **Report Generator:** Produces formatted results and error statuses.

No database, web framework, or external service is required for the initial implementation.

We will use pytest for automated testing.

### Rationale

- Python provides built-in support for text processing and file handling.
- A modular monolith keeps deployment and development simple.
- Separating components improves maintainability and testability.
- The CLI matches the input/output-oriented nature of the original kata.
- Text files are sufficient because the kata does not require persistent application state.
- Avoiding unnecessary infrastructure reduces implementation complexity.

### Consequences

**Positive consequences:**

- Simple local execution and deployment.
- Minimal external dependencies.
- Independent unit testing of individual components.
- Clear separation of responsibilities.
- Easy integration with GitHub Actions and automated tests.

**Negative consequences:**

- The application does not initially provide a graphical or web interface.
- Components are deployed together rather than independently.
- The architecture is not designed for distributed processing.
- Adding persistent storage or a web API would require additional architectural decisions.

### Alternatives Considered

**Microservices architecture:** Rejected because independent deployment and network communication would introduce unnecessary complexity.

**Serverless architecture:** Rejected for the initial implementation because cloud infrastructure is not required for local batch processing.

**Web application:** Deferred because the original Bank OCR kata can be completed through a command-line interface.

### Future Considerations

The modular structure should allow a web API or graphical interface to be added later without rewriting the core recognition and validation logic.
