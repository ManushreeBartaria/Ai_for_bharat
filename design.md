# RepoMind - Design Document (v1)

## System Architecture

### High-Level Overview

RepoMind follows a pipeline architecture with four main stages:

```
GitHub Repo → Parser → Graph Builder → Query Engine → LLM Interface
                                ↓
                          Graph Database
                                ↓
                    Impact Analyzer + Starred Messages
```

### Design Philosophy

- Best-effort analysis: Continue processing even when some files fail
- Static analysis focus: Rely on AST parsing, not runtime behavior
- Heuristic approach: Provide useful approximations rather than guarantees
- Progressive disclosure: Load and display data incrementally

### Component Breakdown

#### 1. Repository Parser
**Responsibility:** Clone and parse repository files

**Subcomponents:**
- **Git Cloner:** Fetches repository from GitHub
- **Language Detector:** Identifies programming languages used
- **AST Parser:** Generates abstract syntax trees for each file
- **Metadata Extractor:** Extracts file structure, imports, and definitions

**Technology Stack:**
- `GitPython` for cloning
- Tree-sitter for multi-language AST parsing
- Language-specific parsers as fallback (e.g., `ast` for Python, `@babel/parser` for JavaScript)

**Error Handling:**
- Skip files that fail parsing
- Log parsing errors for observability
- Continue with partial results

**Output:** Structured representation of successfully parsed files with AST nodes

---

#### 2. Graph Builder
**Responsibility:** Construct multiple interconnected graphs

**Subcomponents:**

##### 2.1 File Graph Builder
- Creates nodes for files and directories
- Establishes parent-child relationships
- Attempts to identify entry points using heuristics:
  - Common patterns: `main.py`, `index.js`, `app.py`, `server.js`
  - Package.json `main` field
  - Setup.py entry points
- Tags configuration files (package.json, requirements.txt, etc.)

**Schema:**
```
FileNode {
  id: string
  path: string
  type: file | directory
  language: string
  size: number
  children: [FileNode]
}
```

##### 2.2 Function/Class Graph Builder
- Extracts function and class definitions where possible
- Builds call graph using static analysis (AST traversal)
- Tracks inheritance hierarchies when detectable
- Links definitions to their file locations

**Limitations:**
- Dynamic calls (e.g., `getattr()`, `eval()`) not captured
- Indirect references may be missed
- Cross-language calls not supported
- Reflection and metaprogramming not analyzed

**Schema:**
```
FunctionNode {
  id: string
  name: string
  file: FileNode
  parameters: [Parameter]
  returnType: string
  calls: [FunctionNode]
  calledBy: [FunctionNode]
}

ClassNode {
  id: string
  name: string
  file: FileNode
  methods: [FunctionNode]
  inherits: [ClassNode]
  inheritedBy: [ClassNode]
}
```

##### 2.3 Dependency Graph Builder
- Parses import statements from source files
- Resolves internal vs external dependencies using heuristics
- Creates dependency edges between modules
- Records versions from package manifests when available

**Limitations:**
- Dynamic imports may be missed
- Conditional imports not fully tracked
- Version resolution is best-effort

**Schema:**
```
DependencyNode {
  id: string
  name: string
  version: string
  type: internal | external
  importedBy: [FileNode]
  imports: [DependencyNode]
}
```

##### 2.4 Semantic Metadata Builder
- Uses LLM to generate short summaries for files and modules
- Attaches natural-language descriptions to graph nodes
- Enables concept-based querying

**Approach:**
- Batch process files to minimize LLM calls
- Generate summaries for key files (entry points, large modules)
- Store embeddings for semantic search

**Limitations:**
- Summaries are heuristic and may be incomplete
- Token limits constrain context per file
- No guarantee of accuracy

**Schema:**
```
SemanticMetadata {
  nodeId: string
  nodeType: file | function | class
  summary: string
  embedding: [float]
  generatedAt: timestamp
}
```

**Technology Stack:**
- Neo4j for graph storage (simpler deployment than ArangoDB)
- NetworkX for graph algorithms
- OpenAI API or local embedding models for semantic understanding

---

#### 3. Query Engine
**Responsibility:** Process natural language queries and retrieve relevant graph nodes

**Subcomponents:**

##### 3.1 Query Parser
- Converts natural language to structured query
- Identifies query intent (explanation, navigation, impact analysis)
- Extracts key entities (function names, file paths, concepts)

##### 3.2 Graph Retriever
- Executes graph traversal based on parsed query
- Ranks nodes by relevance
- Applies context window limits
- Retrieves starred messages if relevant

##### 3.3 Context Builder
- Assembles retrieved nodes into coherent context
- Includes code snippets where necessary
- Adds relationship information
- Formats for LLM consumption

**Query Types:**
- **Explanation:** "How does authentication work?"
- **Navigation:** "Where is function X defined?"
- **Dependency:** "What depends on module Y?"
- **Impact:** "What breaks if I change Z?"

**Technology Stack:**
- Cypher (Neo4j) for graph queries
- Sentence transformers for semantic search (e.g., `all-MiniLM-L6-v2`)
- Simple prompt templates (avoid heavy orchestration frameworks initially)

---

#### 4. Impact Analyzer
**Responsibility:** Estimate downstream effects of proposed changes

**Algorithm:**
1. Identify target node (function, class, or file)
2. Perform breadth-first traversal of dependency graph
3. Collect all downstream nodes within N hops (default: 3, configurable)
4. Categorize impacts:
   - Direct callers
   - Importing files
   - Transitive dependencies
5. Estimate impact level based on proximity to entry points
6. Generate human-readable report

**Impact Level Heuristic:**
```
- Low: Only local references (same file or module)
- Medium: Cross-module references
- High: References near detected entry points or many transitive deps
```

**Confidence Estimate:**
- Based on traversal depth and completeness of call graph
- Lower confidence for dynamic languages or incomplete parsing

**Limitations:**
- Static analysis only; runtime behavior not captured
- May miss dynamic references
- No guarantee of correctness

**Output Format:**
```json
{
  "target": "function_name",
  "impactLevel": "medium",
  "confidence": "moderate",
  "directCallers": ["func1", "func2"],
  "importingFiles": ["module1.py", "module2.py"],
  "transitiveReferences": 12,
  "nearEntryPoints": false,
  "note": "Impact analysis is heuristic and based on static relationships only"
}
```

---

#### 5. Starred Messages System
**Responsibility:** Persist and retrieve important user insights

**Storage Schema:**
```json
{
  "id": "uuid",
  "userId": "string",
  "repoId": "string",
  "message": "string",
  "context": {
    "query": "string",
    "relatedNodes": ["node_ids"]
  },
  "tags": ["architecture", "auth"],
  "timestamp": "ISO8601",
  "embedding": [float]
}
```

**Features:**
- Simple text search across starred messages
- Optional semantic search using embeddings
- Basic relevance ranking for query context

**Technology Stack:**
- PostgreSQL for storage (simpler than MongoDB for v1)
- PostgreSQL full-text search (avoid Elasticsearch complexity initially)
- Sentence transformers for embeddings (optional enhancement)

---

#### 6. LLM Interface
**Responsibility:** Generate natural language responses using graph context

**Workflow:**
1. Receive structured context from Query Engine
2. Inject system prompt with graph-aware instructions
3. Include starred messages if relevant
4. Generate response with citations to graph nodes
5. Format with code snippets and visualizations

**Prompt Template:**
```
You are RepoMind, an AI assistant with deep knowledge of repository structure.

Repository: {repo_name}
Query: {user_query}

Graph Context:
{retrieved_nodes}

Starred References:
{starred_messages}

Provide a clear explanation using the graph structure. Reference specific files and functions.
```

**Technology Stack:**
- OpenAI GPT-4 or Anthropic Claude (configurable)
- Simple prompt templates (avoid framework overhead initially)
- Streaming responses for better UX

---

## Data Flow

### Initialization Flow
1. User provides GitHub URL
2. Repository Parser clones repo to temporary directory
3. AST Parser processes files (skips failures, logs errors)
4. Graph Builder constructs graphs incrementally:
   - File graph (fast)
   - Function/class graph (moderate)
   - Dependency graph (moderate)
   - Semantic metadata (slow, batched)
5. Graphs stored in database
6. System ready for queries (even if some files failed)
7. Cleanup: Remove cloned repo after session timeout

### Query Flow
1. User submits natural language query
2. Query Parser extracts intent and entities
3. Graph Retriever fetches relevant nodes
4. Context Builder assembles LLM input
5. LLM Interface generates response
6. User can star the response
7. Starred message stored with embeddings

### Impact Analysis Flow
1. User specifies target node for change
2. Impact Analyzer traverses dependency graph (limited depth)
3. System collects affected nodes within traversal limit
4. Impact report generated with level estimate and confidence
5. User reviews report (understanding it's heuristic)

---

## Database Schema

### Graph Database (Neo4j)

**Node Types:**
- `File`
- `Directory`
- `Function`
- `Class`
- `Dependency`

**Node Properties:**
- All nodes have `id`, `name`, `type`
- Files have `path`, `language`, `size`, `parseSuccess`
- Functions have `parameters`, `returnType`, `lineNumber`
- Dependencies have `version`, `isExternal`

**Relationship Types:**
- `CONTAINS` (Directory → File)
- `DEFINES` (File → Function/Class)
- `CALLS` (Function → Function)
- `INHERITS` (Class → Class)
- `IMPORTS` (File → Dependency)
- `HAS_METADATA` (File/Function → SemanticMetadata)

**Example Cypher Query:**
```cypher
// Find all functions that call target_function
MATCH (caller:Function)-[:CALLS]->(target:Function {name: 'authenticate'})
RETURN caller.name, caller.file
```

### Relational Database (PostgreSQL)

**Tables:**
- `repositories` (id, url, name, last_updated, parse_status, node_count)
- `starred_messages` (id, repo_id, message, context_json, timestamp)
- `sessions` (id, repo_id, created_at, last_active, status)
- `parse_errors` (id, repo_id, file_path, error_message, timestamp)

---

## API Design

### REST Endpoints

```
POST /api/repos
Body: { "githubUrl": "https://github.com/user/repo" }
Response: { "repoId": "uuid", "status": "processing" }

GET /api/repos/{repoId}/status
Response: { 
  "status": "ready" | "processing" | "partial",
  "graphStats": {
    "totalFiles": 1234,
    "parsedFiles": 1200,
    "failedFiles": 34,
    "functionCount": 5678,
    "dependencyCount": 89
  }
}

POST /api/repos/{repoId}/query
Body: { "query": "How does auth work?" }
Response: { "answer": "...", "sources": [...] }

POST /api/repos/{repoId}/impact
Body: { "target": "function_name", "targetType": "function" }
Response: { 
  "impactLevel": "medium",
  "confidence": "moderate",
  "affectedNodes": [...],
  "note": "Heuristic analysis based on static relationships"
}

POST /api/starred
Body: { "repoId": "uuid", "message": "...", "context": {...} }
Response: { "starredId": "uuid" }

GET /api/starred?repoId={repoId}
Response: { "messages": [...] }
```

---

## Technology Stack Summary

| Component | Technology |
|-----------|-----------|
| Backend | Python (FastAPI) |
| Graph Database | Neo4j Community Edition |
| Relational DB | PostgreSQL |
| AST Parsing | Tree-sitter + language-specific parsers |
| LLM | OpenAI GPT-4 (configurable) |
| Embeddings | Sentence Transformers (all-MiniLM-L6-v2) |
| Frontend | React + D3.js (for visualization) |
| Deployment | Docker Compose (v1), Kubernetes (future) |
| Task Queue | Celery + Redis (for async repo processing) |

---

## Scalability Considerations

### Repository Size Limits
- Target: repos up to ~10k files
- Hard limit: ~20k files (reject larger repos in v1)
- Graph size: up to ~200k nodes

### Traversal Limits
- Default depth: 3 hops for impact analysis
- Configurable max: 5 hops
- Timeout: 15 seconds for traversals

### Caching Strategy
- Cache parsed ASTs temporarily
- Cache query results for 5 minutes
- No persistent cache in v1 (simplicity over performance)

### Incremental Updates
- Not supported in v1
- Full re-parse required for updates
- Future: detect changed files via git diff

---

## Security & Privacy

- Only support public repositories (v1)
- No code execution or arbitrary command running
- Clone to isolated temporary directories
- Automatic cleanup after session timeout (1 hour)
- Rate limiting on API endpoints
- No user authentication in v1 (single-user or demo mode)
- Sanitize all user inputs
- Limit LLM token usage per session

---

## Future Enhancements (Post-v1)

- Private repository support with OAuth
- User authentication and multi-user support
- Incremental graph updates
- IDE plugins (VS Code, JetBrains)
- Multi-repository analysis
- Historical change tracking
- Improved accuracy for dynamic languages
- Support for more languages (Rust, C++, Ruby)
- Real-time collaboration features
- Integration with CI/CD pipelines

## Known Limitations (v1)

- Static analysis only; dynamic behavior not captured
- Best-effort parsing; some files may fail
- Impact analysis is heuristic, not guaranteed
- Single repository per session
- No persistent user accounts
- Limited to ~10k files per repository
- Semantic metadata may be incomplete
- No support for monorepos or multi-language projects with complex build systems
