# RepoMind – Requirements Document (v1)

## Project Overview

RepoMind is an AI-powered system designed to help developers quickly understand and explore GitHub repositories. It reduces the time required for onboarding and open-source contribution by explaining the codebase in a structured, graph-based way. The complete repository is converted into connected graphs using NetworkX, making the codebase machine-understandable.

**Primary Goal:** Reduce manual code navigation by offering graph-based repository understanding with AI-powered explanations and impact analysis.

**Key Innovation:** RepoMind builds real graph representations of the repository instead of only doing text summarization, connecting the entire codebase into File Graph + Function/Class Graph + Dependency Graph with who-calls-who relationships.

## Target Users

- Open-source contributors
- First-time repository explorers
- Students onboarding into OSS projects
- Hackathon teams reusing public code
- Developers reviewing unfamiliar codebases

## Design Principles

- Graph-based repository understanding using NetworkX
- Lightweight local storage using Pickle (.pkl) - no database required
- DFS + BFS hybrid traversal for explanation generation
- DFS-based impact analysis for change propagation
- Gemini 3 Flash Preview + RAG powered repository Q&A
- Who-calls-who function call tracing
- Focus on public open-source repositories

---

## Functional Requirements

### FR1: Repository Ingestion

- Accept public GitHub repository URLs
- Clone repository locally for analysis
- Parse directory and file structure
- Support common languages (Python, JavaScript, TypeScript, Java) on a best-effort basis
- Skip binary files and unsupported formats

### FR2: Graph Construction

RepoMind builds connected graphs using NetworkX where:
- **Nodes** represent: Files, Directories, Functions, Classes
- **Edges** represent: File containment, Import/dependency links, Function calls (who calls who), Class relationships (inheritance/usage)

All graphs are stored locally in Pickle (.pkl) format for fast reuse and lightweight storage.

#### FR2.1 File Graph
- Represent directory hierarchy as graph nodes
- Track file locations and relationships
- Identify common configuration files
- Attempt to locate likely entry points

#### FR2.2 Function/Class Graph
- Extract function and class definitions using Python AST and Tree-sitter
- Build call graph showing who-calls-who relationships
- Track class inheritance and usage patterns
- Support cross-file references when symbols are resolvable
- **Note:** Dynamic behavior may not be fully captured

#### FR2.3 Dependency Graph
- Parse import statements to build dependency edges
- Identify internal module dependencies
- List external libraries when declared
- Record dependency versions when available

#### FR2.4 Call Graph (Who Calls Who)
- Trace function invocations across the codebase
- Build edges showing caller → callee relationships
- Enable traversal to understand execution flow
- Support impact analysis by following call chains

### FR3: Natural Language Query Interface

- Accept user questions in natural language
- Use embeddings + RAG retrieval to extract relevant graph context
- Perform DFS + BFS hybrid traversal to follow execution and dependency paths
- Generate responses using Gemini 3 Flash Preview with retrieved context
- Maintain conversation history during a session

### FR4: Impact Analysis

Given a selected file, function, or class:
- Use DFS traversal to identify affected components
- Identify direct callers through call graph edges
- Identify importing files through dependency graph
- List downstream references within configurable depth
- **Note:** Impact analysis uses DFS on graph structure and is based on static relationships only

#### FR4.1 Impact Reporting
- Show affected files and functions
- Indicate approximate impact level:
  - **Low:** local references
  - **Medium:** cross-module references
  - **High:** references near detected entry points
- Provide basic confidence estimate derived from dependency depth
- **Note:** No guarantees of correctness

### FR5: Starred Responses

- Allow users to star important responses
- Persist starred messages locally using JSON or file-based storage
- Include starred content as optional context in later queries
- Provide simple text search over starred messages
- **Note:** No database required - uses local file storage

### FR6: Visualization

- Display file and dependency graphs interactively
- Show who-calls-who relationships in call graph
- Allow node expansion and collapse
- Highlight impact-analysis paths using DFS traversal results
- Use progressive loading for larger graphs

### FR7: Entry Point Detection

Attempt to identify:
- Main application files
- Server startup scripts
- CLI entry points

**Note:** Detection is heuristic and framework-dependent

### FR8: Auto Documentation Generator

- Generate structured repository documentation automatically
- Use Gemini 3 Flash Preview to create summaries
- Include architecture overview, key components, and dependencies
- Export documentation in markdown format

---

## Non-Functional Requirements

### NFR1: Performance

**Repository parsing:**
- < 2 minutes for repos under 2k files
- < 5 minutes for repos under 10k files

**Graph storage:**
- Pickle (.pkl) files for fast serialization/deserialization
- No database overhead

**Query response:**
- < 30 seconds for cached queries
- < 1 min for uncached queries with RAG retrieval

**Impact analysis:**
- < 30 seconds for DFS traversals

### NFR2: Scalability

- Support graphs up to ~200k nodes
- Limit traversal depth by default
- Support incremental re-indexing
- Single-repository processing per session

### NFR3: Accuracy

- Best-effort dependency detection using static analysis
- Call graphs built using Python AST and Tree-sitter
- Who-calls-who relationships limited to statically analyzable code
- Impact analysis uses DFS and favors recall over precision

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
- Cap Gemini API token usage
- Timeout long-running DFS/BFS traversals
- Remove inactive sessions
- Use local Pickle storage to avoid database costs

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
- Starred responses used in at least 30% of sessions
- Queries return relevant context using RAG retrieval
- Graph-based understanding provides deeper insights than text-based approaches
- DFS/BFS traversal accurately follows code execution paths

---

## Out of Scope (v1)

- Private repositories
- IDE plugins
- Automatic code modification
- Multi-repository analysis
- Real-time collaboration
- Historical change tracking
- User authentication
