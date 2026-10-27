# CodeSense — AI Codebase Comprehension & PR Review Copilot

> An AI-powered developer tool that helps engineers understand unfamiliar codebases, analyze pull requests in context, identify potential defects, and recommend targeted tests.

**Track:** AI/ML  
**Target:** Java repositories for the initial MVP  
**Project status:** MVP in development

---

## 1. Problem Statement

Large software repositories are difficult to understand, especially for developers joining an unfamiliar project. Understanding architecture, tracing dependencies, reviewing pull requests, and identifying regression risks often require substantial manual effort.

Traditional code search finds files and symbols, while an LLM reviewing only a pull-request diff may miss important surrounding context.

CodeSense aims to combine structural code analysis, semantic retrieval, and large language models to provide repository-aware explanations and pull-request reviews grounded in actual source code.

## 2. Proposed Solution

CodeSense indexes a repository using a skeleton-first approach. It first extracts lightweight structural information and then retrieves relevant source code on demand.

When a developer submits a pull request, CodeSense analyzes the changes, identifies related code and tests, and generates an evidence-backed review with potential risks and test recommendations.

### Core capabilities

1. **Skeleton-first repository indexing**
   - Import a public GitHub repository.
   - Filter irrelevant, generated, binary, and oversized files.
   - Extract file structure, Java classes, methods, imports, and basic relationships.
   - Store searchable code chunks and metadata.

2. **Architecture Q&A**
   - Answer natural-language questions about the repository.
   - Retrieve relevant code using semantic search.
   - Provide file and line references.
   - Indicate when retrieved evidence is insufficient.

3. **Context-aware pull-request review**
   - Fetch PR metadata and changed-file patches.
   - Identify changed classes and methods.
   - Retrieve relevant callers, related classes, and tests.
   - Flag potential logic errors, security issues, and convention violations.
   - Explain findings using supporting source evidence.

4. **Test recommendations**
   - Identify relevant existing tests.
   - Recommend targeted test scenarios based on changed behavior.
   - Optionally generate JUnit test examples.

**MVP scope:** Java-first repository analysis, grounded architecture Q&A, pull-request review, and test recommendations. Autonomous repair, automatic merging, and broad multi-language support are future extensions.

---

## 3. Technology Stack

| Layer | Technology | Responsibility |
|---|---|---|
| Backend | Java 21, Spring Boot | REST APIs and application logic |
| Code parsing | JavaParser | Java classes, methods, imports, and method-call analysis |
| AI orchestration | Spring AI | LLM and embedding integration |
| Vector search | PostgreSQL + pgvector | Semantic code retrieval |
| Database access | Spring Data JPA / JDBC | Metadata persistence |
| Repository integration | GitHub REST API or Git | Repository and PR retrieval |
| Frontend | React + Vite | Repository input, chat, and review UI |
| Streaming | Server-Sent Events (SSE) | Stream chat responses if supported by the existing backend |
| Testing | JUnit 5, Mockito | Unit and integration testing |
| Build | Maven | Dependency management and Java builds |
| Optional static analysis | Semgrep | Additional supported code and security checks |

Python is not required for the initial MVP. The core implementation can remain in Java.

---

## 4. System Architecture

```text
React + Vite
     |
     v
Spring Boot REST API
     |
     +--> Repository Ingestion
     |       |
     |       +--> GitHub API / Git
     |       +--> File Filtering
     |       +--> JavaParser
     |       +--> Skeleton / Symbol Index
     |       +--> Chunking and Embeddings
     |
     +--> Architecture Q&A
     |       |
     |       +--> Semantic Retrieval
     |       +--> Structural Context Expansion
     |       +--> LLM Answer Generation
     |
     +--> Pull Request Analysis
             |
             +--> GitHub PR Metadata and Diff
             +--> Changed Method Identification
             +--> Impact Analysis
             +--> Related Code and Test Retrieval
             +--> Review Engine
             +--> Test Recommendations

Shared storage:
PostgreSQL + pgvector
```

### Key design principle

Structural analysis and semantic search have different responsibilities:

- **Structural analysis** identifies known code relationships, such as method declarations and detected method calls.
- **Semantic search** retrieves code that is relevant in meaning.
- **The LLM** reasons over the diff and retrieved context to explain potential issues.

Semantic similarity alone does not prove that one method depends on another. Cross-file call resolution may also be incomplete, so findings should reflect the evidence actually available.

---

## 5. Module-by-Module Implementation

### Module 1: Repository Ingestion and Indexing

**Suggested package:** `repository` and `indexing`

Implementation steps:

1. Accept and validate a public GitHub repository URL.
2. Resolve the repository owner and name.
3. Fetch the selected commit and repository file tree.
4. Filter binary, generated, dependency, and oversized files.
5. Build a lightweight file and symbol map.
6. Use JavaParser to extract supported Java structures.
7. Reuse the existing code chunking and embedding pipeline.
8. Store code chunks, repository metadata, commit SHA, and structural information.
9. Expose indexing status and error details.

**Acceptance criteria**

- A valid public repository can be imported.
- The selected commit SHA is recorded.
- Files are filtered and indexed without duplicating existing functionality.
- Relevant source code can be retrieved by the existing search pipeline.

### Module 2: Architecture Q&A

**Suggested package:** `rag`

Implementation steps:

1. Accept a repository identifier and user question.
2. Generate the question embedding through the configured provider.
3. Retrieve relevant chunks from pgvector.
4. Optionally expand context with related structural information.
5. Build a prompt using retrieved code and source metadata.
6. Generate an answer using the existing LLM integration.
7. Return file paths and line references where available.
8. Return an insufficient-evidence response when the context does not support an answer.

**Acceptance criteria**

A question about a real repository produces a relevant answer with verifiable source references.

### Module 3: Pull-Request Ingestion

**Suggested package:** `pullrequest`

Implementation steps:

1. Accept a GitHub pull-request URL.
2. Validate the GitHub host, repository path, and PR number.
3. Fetch PR metadata, base/head SHAs, changed files, and patches.
4. Parse the diff into added, deleted, and modified lines.
5. Map supported changed lines to Java classes and methods.
6. Handle missing patches, deleted files, and unsupported files gracefully.

**Acceptance criteria**

A valid PR produces a structured change set containing file paths, change details, and commit identifiers.

### Module 4: Change-Impact Analysis

**Suggested package:** `impact`

Implementation steps:

1. Identify methods and classes affected by the diff.
2. Find known callers and related symbols in the structural index.
3. Retrieve relevant tests, configuration, and related code.
4. Use semantic search to find additional relevant context.
5. Record the evidence for each reported relationship.
6. Distinguish confirmed structural links from possible semantic associations.

**Acceptance criteria**

The review report identifies relevant affected components with supporting evidence for the chosen demo PR.

### Module 5: Review Engine

**Suggested package:** `review`

Implementation steps:

1. Collect the diff, changed methods, and relevant context.
2. Ask the configured LLM to identify potential defects and architectural deviations.
3. Request structured findings containing severity, explanation, source location, evidence, and a suggested fix.
4. Optionally incorporate static-analysis results.
5. Deduplicate findings and validate source locations.
6. Mark uncertain findings as potential issues rather than confirmed defects.

**Acceptance criteria**

A deliberately flawed PR produces a credible, explainable finding supported by the source code.

### Module 6: Test Recommendations

**Suggested package:** `testing`

Implementation steps:

1. Consume review findings and impact-analysis results.
2. Retrieve related existing tests.
3. Identify missing boundary cases and failure scenarios.
4. Recommend targeted test cases.
5. Optionally generate JUnit test snippets.

**Acceptance criteria**

Test recommendations are connected to the changed behavior and the potential issue. Test execution is an optional extension.

### Module 7: React UI

Implementation steps:

1. Create a repository import form.
2. Display indexing status and errors.
3. Provide an architecture Q&A interface.
4. Create a PR URL input and analysis action.
5. Display findings with severity, evidence, and source locations.
6. Show affected components and test recommendations.
7. Handle loading, empty, and failure states.

**Acceptance criteria**

A user can complete the MVP workflow through the UI without manually invoking backend endpoints.

---

## 6. Proposed API Design

These are proposed endpoints. Reuse existing routes where equivalent functionality already exists.

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/repositories/import` | Import and index a repository |
| GET | `/api/repositories/{id}/status` | Retrieve indexing status |
| POST | `/api/chat` | Ask a repository question |
| POST | `/api/pull-requests/analyze` | Analyze a PR URL |
| GET | `/api/reviews/{id}` | Retrieve a review report |

Example repository import request:

```json
{
  "repositoryUrl": "https://github.com/owner/repository"
}
```

Example PR analysis request:

```json
{
  "pullRequestUrl": "https://github.com/owner/repository/pull/42"
}
```

Example review response shape:

```json
{
  "repository": "owner/repository",
  "baseSha": "base-commit-sha",
  "headSha": "head-commit-sha",
  "findings": [
    {
      "title": "Potential retry-state inconsistency",
      "severity": "MEDIUM",
      "filePath": "src/main/java/example/PaymentService.java",
      "line": 85,
      "explanation": "The changed branch may leave retry metadata inconsistent with the terminal state.",
      "evidence": [
        "Changed retry-limit branch",
        "Related scheduler selection logic"
      ],
      "suggestedTests": [
        "Verify terminal status and retry timestamp",
        "Verify the scheduler excludes terminal payments"
      ]
    }
  ]
}
```

The example is illustrative; real findings must be derived from actual retrieved code. Production implementations should use validated DTOs and stable response schemas.

---

## 7. Database Design

Reuse existing tables wherever possible. A possible logical schema is:

### `repositories`

- `id`
- `owner`
- `name`
- `repository_url`
- `indexed_commit_sha`
- `status`
- `error_message`
- `created_at`
- `updated_at`

### `indexed_files`

- `id`
- `repository_id`
- `commit_sha`
- `file_path`
- `language`
- `file_type`
- `content_hash`

### `code_symbols`

- `id`
- `repository_id`
- `commit_sha`
- `file_path`
- `symbol_name`
- `symbol_type`
- `start_line`
- `end_line`
- `signature`

### `code_relationships`

- `id`
- `repository_id`
- `commit_sha`
- `source_symbol_id`
- `target_symbol_id`
- `relationship_type`
- `confidence`

Only store relationships that the parser or analysis actually establishes. Unknown or unresolved calls should not be presented as confirmed dependencies.

### `code_chunks`

Store chunk text, repository and commit identifiers, file path, line range, and embedding. The precise vector column and index depend on the embedding model's dimensions and the existing pgvector schema.

### `review_reports`

- `id`
- `repository_id`
- `pull_request_number`
- `base_sha`
- `head_sha`
- `status`
- `report_json`
- `created_at`

This schema is a proposed design, not a claim that these tables already exist.

---

## 8. Configuration and Environment

Keep secrets in environment variables or an appropriate secret manager. Do not commit tokens, database passwords, or API keys.

Illustrative environment variables:

```properties
SPRING_DATASOURCE_URL=${DATABASE_URL}
SPRING_DATASOURCE_USERNAME=${DATABASE_USERNAME}
SPRING_DATASOURCE_PASSWORD=${DATABASE_PASSWORD}

# Configure these using the actual provider and
# Spring AI dependencies in the project.
GITHUB_TOKEN=${GITHUB_TOKEN}
```

The exact property names for the AI provider depend on the Spring AI version and provider already used by the project.

Before adding dependencies, inspect `pom.xml` and the existing configuration files. Avoid maintaining conflicting `application.yml` and `application.properties` values for the same settings.

---

## 9. Running and Testing

The commands below assume a Maven-based Spring Boot project with the Maven Wrapper.

### Run backend tests

```bash
./mvnw test
```

On Windows PowerShell:

```powershell
.\mvnw.cmd test
```

### Start the backend

```powershell
.\mvnw.cmd spring-boot:run
```

The database and required environment variables must be configured first.

### Start the frontend

From the frontend directory:

```bash
npm install
npm run dev
```

The actual frontend directory and npm scripts must match the repository.

### Minimum validation checklist

- Repository URL validation works.
- Indexing records the correct commit SHA.
- Generated and irrelevant files are filtered.
- Retrieval returns the expected source files.
- Q&A references valid source locations.
- PR ingestion handles missing or unsupported patches.
- Findings refer to actual changed or related code.
- Test recommendations match the reported behavior.
- Frontend displays errors rather than silently failing.
- No secrets are committed to the repository.

---

## 10. Security and Reliability

- Validate GitHub URLs and reject unexpected hosts.
- Avoid leaking tokens or secrets found in repository content.
- Isolate repository data by repository ID and commit.
- Treat source code, PR descriptions, comments, and retrieved files as untrusted input.
- Do not allow repository instructions to override system-level review rules.
- Do not execute arbitrary repository code on the main backend.
- If test execution is added, use an isolated environment with resource limits and restricted network access.
- Keep human review in the approval and merge process.

---

## 11. Twenty-Four-Hour Hackathon Plan

| Time | Goal |
|---|---|
| Hours 0–2 | Audit the existing indexing, RAG, database, and UI |
| Hours 2–6 | Complete repository ingestion and structural indexing |
| Hours 6–8 | Verify grounded architecture Q&A |
| Hours 8–10 | Implement PR retrieval and diff parsing |
| Hours 10–13 | Implement basic impact analysis |
| Hours 13–16 | Implement evidence-backed review findings |
| Hours 16–18 | Add test recommendations |
| Hours 18–21 | Integrate the frontend and validate the workflow |
| Hours 21–24 | Fix critical bugs, test the seeded defect, and prepare the demo |

Adjust this plan based on what already works. If the existing RAG pipeline is broken, prioritize a reliable end-to-end workflow over advanced structural analysis.

### Out of scope for the MVP

- Autonomous multi-agent repair
- Automatic merging or approving pull requests
- Full multi-language parsing
- Perfect whole-program dependency resolution
- Running arbitrary code without isolation
- Claims of production-grade bug detection

---

## 12. Demonstration Plan

Use a real open-source Java repository and a deliberately flawed pull request.

1. Import and index the repository.
2. Ask an architecture question and show the grounded answer.
3. Submit the flawed PR URL.
4. Show the changed method and related code.
5. Display a potential defect with supporting evidence.
6. Show the affected component and targeted test recommendation.

Evaluate whether the seeded issue is detected, whether the evidence is correct, and whether false positives occur. Do not claim universal detection accuracy from a small demo.

---

## 13. Project Success Criteria

The MVP is successful when:

- A public repository can be indexed at a known commit.
- Architecture questions return source-grounded answers.
- A PR URL produces a structured review report.
- At least one known defect is correctly identified in the demo.
- Related code or tests are retrieved with verifiable evidence.
- Targeted test recommendations are shown.
- The workflow operates through the React interface.

## 14. Future Enhancements

- GitHub webhooks for automatic review on PR updates
- Inline review comments through GitHub APIs
- More programming languages using Tree-sitter
- Deeper call-graph resolution
- Static-analysis integration
- Isolated test execution and result reporting
- Human-approved patch generation
- Carefully scoped multi-agent repair workflows

---

**CodeSense's core differentiator:** a traceable change-impact workflow that combines repository structure and semantic retrieval to explain why a code change may affect other components and what should be tested.

The objective is not to replace human reviewers, but to give them better context and evidence when reviewing unfamiliar codebases.
