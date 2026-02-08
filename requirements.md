# RepoMind – Requirements Document (v1)

## Project Overview

RepoMind is a repository analysis tool that processes public GitHub repositories and builds structured graphs to support codebase exploration. The system provides basic architectural insights, dependency tracing, and change impact estimation to assist contributors working with unfamiliar open-source projects.

**Primary Goal:** Reduce manual code navigation by offering structured context and query-based exploration.

## Target Users

- Open-source contributors
- First-time repository explorers
- Students onboarding into OSS projects
- Hackathon teams reusing public code
- Developers reviewing unfamiliar codebases

## Design Principles

- Prefer structured analysis over raw text search
- Prioritize clarity over automation
- Provide best-effort results where static analysis is limited
- Focus on public open-source repositories
- Avoid assumptions about developer intent

---

## Functional Requirements

### FR1: Repository Ingestion

- Accept public GitHub repository URLs
- Clone repository locally for analysis
- Parse directory and file structure
- Support common languages (Python, JavaScript, TypeScript, Java) on a best-effort basis
- Support repositories up to approximately 10k files
- Skip binary files and unsupported formats

### FR2: Graph Construction

#### FR2.1 File Graph
- Represent directory hierarchy
- Track file locations
- Identify common configuration files
- Attempt to locate likely entry points

#### FR2.2 Function/Class Graph
- Extract function and class definitions where possible
- Build basic call relationships using static analysis
- Track class inheritance when detectable
- Support cross-file references when symbols are resolvable
- **Note:** Dynamic behavior may not be fully captured

#### FR2.3 Dependency Graph
- Parse import statements
- Identify internal module dependencies
- List external libraries when declared
- Record dependency versions when available

#### FR2.4 Semantic Metadata
- Store short summaries of files and modules generated via LLM
- Attach natural-language descriptions to graph nodes
- Enable basic concept-based querying
- **Note:** This metadata is heuristic and may be incomplete

### FR3: Natural Language Query Interface

- Accept user questions in natural language
- Retrieve relevant graph nodes
- Generate responses using LLM and retrieved context
- Maintain conversation history during a session

### FR4: Impact Analysis

Given a selected file, function, or class:
- Identify direct callers
- Identify importing files
- List downstream references within configurable depth
- **Note:** Impact analysis is heuristic and based on static relationships only

#### FR4.1 Impact Reporting
- Show affected files and functions
- Indicate approximate impact level:
  - **Low:** local references
  - **Medium:** cross-module references
  - **High:** references near detected entry points
- Provide basic confidence estimate derived from dependency depth
- **Note:** No guarantees of correctness

### FR5: Starred Messages

- Allow users to star responses
- Persist starred messages per repository
- Include starred content as optional context in later queries
- Provide simple text search over starred messages

### FR6: Visualization

- Display file and dependency graphs
- Allow node expansion and collapse
- Highlight impact-analysis paths
- Use progressive loading for larger graphs

### FR7: Entry Point Detection

Attempt to identify:
- Main application files
- Server startup scripts
- CLI entry points

**Note:** Detection is heuristic and framework-dependent

---

## Non-Functional Requirements

### NFR1: Performance

**Repository parsing:**
- < 2 minutes for repos under 2k files
- < 5 minutes for repos under 10k files

**Query response:**
- < 30 seconds for cached queries
- < 1 min for uncached queries

**Impact analysis:**
- < 30 seconds for typical traversals

### NFR2: Scalability

- Support graphs up to ~200k nodes
- Limit traversal depth by default
- Support incremental re-indexing
- Single-repository processing per session

### NFR3: Accuracy

- Best-effort dependency detection
- Call graphs limited to statically analyzable code
- Impact analysis favors recall over precision

### NFR4: Usability

- Basic natural language interface
- Simple visual exploration
- Readable impact listings
- Minimal interaction required for starring content

### NFR5: Reliability

- Continue analysis if some files fail parsing
- Skip unsupported languages
- Log failures
- Allow partial graph generation

### NFR6: Security

- Public repositories only
- No authentication storage
- Temporary local cloning
- Automatic cleanup after session ends

### NFR7: Observability

- Log ingestion failures
- Track parsing duration
- Record query latency
- Capture impact analysis execution time

### NFR8: Cost Control

- Limit graph size per session
- Cap LLM token usage
- Timeout long-running traversals
- Remove inactive sessions

---

## Use Cases

### UC1: Repository Exploration
User provides a GitHub URL and asks general questions about structure or functionality.

### UC2: Change Review
User selects a function and requests impact analysis to view dependent files.

### UC3: Dependency Lookup
User queries where a module is imported or used.

### UC4: Learning from Open Source
User explores components and stars explanations for later reference.

---

## Success Metrics

- Reduced manual file searching during sessions
- Users able to identify dependencies without external tools
- Starred messages used in at least 30% of sessions
- Queries return relevant context most of the time

---

## Out of Scope (v1)

- Private repositories
- IDE plugins
- Automatic code modification
- Multi-repository analysis
- Real-time collaboration
- Historical change tracking
- User authentication
