# RepoMind - Design Document (v1)

## System Architecture

### High-Level Overview

RepoMind follows a lightweight pipeline architecture with graph-based repository understanding:

```
GitHub Repo → Parser → Graph Builder (NetworkX) → Graph Storage (Pickle) 
                                                         ↓
                    DFS+BFS Traversal ← RAG Retrieval ← Embeddings
                            ↓
                    Gemini 3 Flash Preview → Output Dashboard
                            ↓
                    Starred Messages (Local JSON/File)
```

### Design Philosophy

- Graph-based understanding: Build real graph representations instead of text summarization
- Lightweight architecture: No database required (NetworkX + Pickle storage)
- Who-calls-who tracing: Track function call relationships across the codebase
- DFS + BFS hybrid: Follow execution and dependency paths intelligently
- RAG-powered: Use embeddings to retrieve relevant graph context
- Best-effort analysis: Continue processing even when some files fail
- Static analysis focus: Rely on Python AST and Tree-sitter parsing

### Component Breakdown

#### 1. Repository Ingestion
**Responsibility:** Clone and download GitHub repositories

**Subcomponents:**
- **Git Cloner:** Fetches repository from GitHub using git clone or py-clone-git
- **Repository Validator:** Checks repository size and accessibility

**Technology Stack:**
- `GitPython` or `git` command-line tool
- FastAPI for backend API

**Error Handling:**
- Validate repository URL before cloning
- Check repository size limits
- Handle private repository errors

**Output:** Local copy of repository ready for parsing

---

#### 2. Code Parsing
**Responsibility:** Extract code structure using static analysis

**Subcomponents:**
- **Python AST Parser:** Parses Python files using built-in `ast` module
- **Tree-sitter Parser:** Multi-language parsing for JavaScript, TypeScript, Java, etc.
- **Language Detector:** Identifies programming languages used
- **Metadata Extractor:** Extracts files, functions, classes, imports, and dependencies

**Technology Stack:**
- Python `ast` module for Python code
- Tree-sitter for multi-language support
- Language-specific parsers as needed

**Error Handling:**
- Skip files that fail parsing
- Log parsing errors for observability
- Continue with partial results

**Output:** Structured representation of files, functions, classes, imports, and dependencies

---

#### 3. Graph Builder (NetworkX)
**Responsibility:** Construct a single unified graph representing the entire repository

RepoMind builds one comprehensive NetworkX MultiDiGraph containing:

**Node Types:**
1. **File nodes** - Files and directories
2. **Function nodes** - Function definitions
3. **Class nodes** - Class definitions
4. **Dependency nodes** - External libraries

**Edge Types (Relationships):**
1. **contains** - Directory contains file
2. **imports** - File imports dependency
3. **calls** - Function calls another function (who-calls-who)
4. **inherits** - Class inherits from another class
5. **defines** - File defines function/class

**Subcomponents:**

##### 3.1 File Graph Builder
- Creates nodes for files and directories in the unified graph
- Node type: 'file' or 'directory'
- Establishes parent-child edges with relationship='contains'
- Attempts to identify entry points using heuristics:
  - Common patterns: `main.py`, `index.js`, `app.py`, `server.js`
  - Package.json `main` field
  - Setup.py entry points
- Tags configuration files (package.json, requirements.txt, etc.)

**NetworkX Schema:**
```python
# Add to unified graph G
G.add_node('file_1', type='file', path='src/main.py', language='python', size=1024, is_entry_point=True)
G.add_node('dir_1', type='directory', path='src/', file_count=10)
G.add_edge('dir_1', 'file_1', relationship='contains')
```

##### 3.2 Function/Class Graph Builder
- Extracts function and class definitions using Python AST and Tree-sitter
- Adds nodes with type='function' or type='class' to unified graph
- Builds call graph showing who-calls-who relationships with relationship='calls' edges
- Tracks inheritance hierarchies with relationship='inherits' edges
- Links definitions to their file locations with relationship='defines' edges
- Stores embeddings as node attributes for semantic search

**Limitations:**
- Dynamic calls (e.g., `getattr()`, `eval()`) not captured
- Indirect references may be missed
- Reflection and metaprogramming not analyzed

**NetworkX Schema:**
```python
# Add to unified graph G
G.add_node('func_1', type='function', name='authenticate', file='file_1', 
           parameters=['username', 'password'], return_type='bool', 
           line_number=45, embedding=[...])

G.add_node('class_1', type='class', name='User', file='file_1', 
           methods=['func_1', 'func_2'], line_number=10, embedding=[...])

# Relationships
G.add_edge('file_1', 'func_1', relationship='defines')
G.add_edge('func_2', 'func_1', relationship='calls')  # func_2 calls func_1
G.add_edge('class_2', 'class_1', relationship='inherits')  # class_2 inherits from class_1
```

##### 3.3 Dependency Graph Builder
- Parses import statements from source files
- Adds nodes with type='dependency' to unified graph
- Resolves internal vs external dependencies using heuristics
- Creates dependency edges with relationship='imports'
- Records versions from package manifests when available

**Limitations:**
- Dynamic imports may be missed
- Conditional imports not fully tracked
- Version resolution is best-effort

**NetworkX Schema:**
```python
# Add to unified graph G
G.add_node('dep_1', type='dependency', name='requests', version='2.28.0', 
           is_external=True)
G.add_edge('file_1', 'dep_1', relationship='imports')
```

**Technology Stack:**
- NetworkX MultiDiGraph for unified graph construction and storage
- Python AST for Python parsing
- Tree-sitter for multi-language parsing

---

#### 4. Graph Storage (Pickle)
**Responsibility:** Persist single unified graph locally for fast reuse

**Approach:**
- Serialize single NetworkX MultiDiGraph using Python Pickle
- Store as one `.pkl` file per repository
- Lightweight storage with no database overhead
- Embeddings stored as node attributes within the graph

**Storage Format:**
```python
import pickle
import networkx as nx

# Save single unified graph
with open('.repomind/graphs/my_repo_graph.pkl', 'wb') as f:
    pickle.dump(G, f)

# Load graph
with open('.repomind/graphs/my_repo_graph.pkl', 'rb') as f:
    G = pickle.load(f)

# Query by node type
file_nodes = [n for n, d in G.nodes(data=True) if d['type'] == 'file']
function_nodes = [n for n, d in G.nodes(data=True) if d['type'] == 'function']

# Query by relationship type
call_edges = [(u, v) for u, v, d in G.edges(data=True) if d['relationship'] == 'calls']
```

**Benefits:**
- Single file for entire repository graph
- Fast serialization and deserialization
- No database installation or configuration required
- Portable storage format
- Easy to version control (optional)
- All node types and relationships in one unified structure

**Technology Stack:**
- Python Pickle for serialization
- Local file system for storage

---

#### 5. DFS + BFS Traversal Engine
**Responsibility:** Navigate graphs to generate explanations and trace execution flow

**Traversal Strategies:**

##### 5.1 DFS (Depth-First Search)
- Used for impact analysis
- Follows dependency chains deeply
- Identifies all affected components when a change is made
- Traces who-calls-who relationships

##### 5.2 BFS (Breadth-First Search)
- Used for exploration and explanation
- Discovers nearby components first
- Provides context around a specific node
- Maps immediate dependencies

##### 5.3 Hybrid DFS + BFS
- Combines both approaches for comprehensive understanding
- BFS for immediate context
- DFS for deep dependency tracing
- Follows actual code connections

**Implementation:**
```python
import networkx as nx

# DFS for impact analysis
def analyze_impact(graph, target_node, max_depth=5):
    affected = []
    for node in nx.dfs_preorder_nodes(graph, target_node, depth_limit=max_depth):
        affected.append(node)
    return affected

# BFS for context gathering
def gather_context(graph, target_node, max_depth=3):
    context = []
    for node in nx.bfs_tree(graph, target_node, depth_limit=max_depth):
        context.append(node)
    return context
```

**Technology Stack:**
- NetworkX built-in DFS and BFS algorithms
- Custom traversal logic for hybrid approach

---

#### 6. Embeddings + RAG Retrieval
**Responsibility:** Extract relevant graph context using semantic search

**Approach:**
1. Generate embeddings for graph nodes (files, functions, classes)
2. Store embeddings alongside graph data
3. When user queries, embed the query
4. Retrieve most relevant nodes using cosine similarity
5. Extract subgraph around relevant nodes
6. Pass to LLM for answer generation

**RAG Pipeline:**
```
User Query → Embed Query → Similarity Search → Retrieve Nodes 
           → DFS/BFS Traversal → Context Assembly → Gemini LLM
```

**Technology Stack:**
- Sentence Transformers for embeddings (e.g., `all-MiniLM-L6-v2`)
- NumPy/SciPy for similarity computation
- NetworkX for subgraph extraction

---

#### 7. Impact Analyzer (DFS-Based)
**Responsibility:** Identify affected components when changes are made

**Algorithm:**
1. Identify target node (function, class, or file)
2. Perform DFS traversal following dependency edges
3. Collect all downstream nodes within N hops (default: 5)
4. Categorize impacts:
   - Direct callers (who calls this function)
   - Importing files (who imports this module)
   - Transitive dependencies (indirect effects)
5. Estimate impact level based on proximity to entry points
6. Generate human-readable report

**DFS Traversal:**
```python
def dfs_impact_analysis(call_graph, target_func, max_depth=5):
    """Find all functions affected if target_func changes"""
    affected = set()
    
    # Find all callers using DFS
    for caller in nx.dfs_preorder_nodes(call_graph.reverse(), 
                                         target_func, 
                                         depth_limit=max_depth):
        affected.add(caller)
    
    return affected
```

**Impact Level Heuristic:**
```
- Low: Only local references (same file or module)
- Medium: Cross-module references
- High: References near detected entry points or many transitive deps
```

**Output Format:**
```json
{
  "target": "authenticate",
  "impactLevel": "high",
  "confidence": "moderate",
  "directCallers": ["login", "verify_token"],
  "importingFiles": ["auth.py", "api.py"],
  "transitiveReferences": 24,
  "affectedFiles": ["auth.py", "api.py", "middleware.py"],
  "note": "DFS-based impact analysis using static call graph"
}
```

---

#### 8. Gemini 3 Flash Preview Integration
**Responsibility:** Generate summaries, documentation, and Q&A responses

**Workflow:**
1. Receive structured context from RAG retrieval
2. Inject system prompt with graph-aware instructions
3. Include relevant code snippets and relationships
4. Generate response using Gemini 3 Flash Preview API
5. Format with citations to graph nodes

**Prompt Template:**
```
You are RepoMind, an AI assistant with deep knowledge of repository structure.

Repository: {repo_name}
Query: {user_query}

Graph Context (from DFS+BFS traversal):
{retrieved_nodes}

Call Graph (who-calls-who):
{call_relationships}

Provide a clear explanation using the graph structure. Reference specific files and functions.
```

**Technology Stack:**
- Google Gemini 3 Flash Preview API
- Simple prompt templates
- Streaming responses for better UX

---

#### 9. Auto Documentation Generator
**Responsibility:** Create structured repository documentation automatically

**Features:**
- Generate architecture overview from graph structure
- List key components and entry points
- Document dependencies and their versions
- Explain major modules and their relationships
- Export as markdown files

**Approach:**
1. Analyze graph structure to identify key components
2. Use Gemini to generate natural language descriptions
3. Include code snippets and diagrams
4. Format as structured markdown

**Output:**
- README-style documentation
- Architecture diagrams (text-based)
- Dependency lists
- API documentation (if applicable)

---

#### 10. Starred Messages System
**Responsibility:** Save important responses locally for reuse

**Storage Approach:**
- Local text file per repository
- Simple file-based storage (no database, no JSON)
- Plain text format for easy reading
- Append-only for new starred messages

**Storage Format:**
```
Repository: user/repo
URL: https://github.com/user/repo

---

[2024-01-15 10:30:00]
Query: How does authentication work?
Answer: Authentication uses JWT tokens and is handled in auth.py...
Related: auth.py, authenticate()
Tags: authentication, security

---
```

**File Location:**
```
.repomind/
  └── starred/
      └── {repo_name}_starred.txt
```

**Technology Stack:**
- Python file I/O for reading/writing
- Local file system
- Plain text format

---

## Data Flow

### Initialization Flow
1. User provides GitHub URL via frontend
2. FastAPI backend receives request
3. Repository Ingestion clones repo to temporary directory
4. Code Parsing extracts structure using Python AST + Tree-sitter
5. Graph Builder constructs single unified NetworkX MultiDiGraph:
   - File nodes (files/directories) with 'contains' edges
   - Dependency nodes (imports) with 'imports' edges
   - Function/Class nodes (definitions) with 'defines' edges
   - Call graph edges (who-calls-who) with 'calls' edges
   - Inheritance edges with 'inherits' edges
6. Embeddings generated and stored as node attributes
7. Single graph serialized to Pickle (.pkl) file
8. System ready for queries
9. Cleanup: Remove cloned repo after processing

### Query Flow
1. User submits natural language query via Repo Chat
2. Query embedded using sentence transformers
3. RAG Retrieval finds relevant nodes by comparing query embedding with node embeddings
4. DFS + BFS Traversal extracts context around relevant nodes from unified graph
5. Context assembled with code snippets and relationships
6. Gemini 3 Flash Preview generates response
7. Response displayed with citations
8. User can star the response (saved to local text file)

### Impact Analysis Flow
1. User selects target node (file/function/class) in Impact page
2. DFS traversal starts from target node in unified graph
3. System follows edges with relationship='calls' (who-calls-who)
4. Collects affected nodes within depth limit
5. Categorizes impact (direct callers, importing files, transitive deps)
6. Estimates impact level (low/medium/high)
7. Generates report with affected components
8. User reviews DFS-based impact analysis

### Graph View Flow
1. User navigates to Graph View page
2. Frontend requests graph data from backend
3. Backend loads single Pickle file containing unified graph
4. NetworkX graph converted to visualization format (nodes and edges)
5. Interactive graph displayed showing:
   - File nodes and directory structure
   - Function/class nodes
   - Who-calls-who edges (relationship='calls')
   - Dependency edges (relationship='imports')
   - Inheritance edges (relationship='inherits')
6. User can expand/collapse nodes and explore relationships
7. Filter by node type (file/function/class/dependency) or relationship type

---

## Storage Architecture

### Graph Storage (Pickle Files)

**Directory Structure:**
```
.repomind/
  └── graphs/
      └── {repo_name}_graph.pkl
  └── starred/
      └── {repo_name}_starred.txt
```

**Graph File:**
- `{repo_name}_graph.pkl` - Single NetworkX MultiDiGraph containing all graph types:
  - File nodes (files and directories)
  - Function nodes (function definitions)
  - Class nodes (class definitions)
  - Dependency nodes (external libraries)
  - Edges with relationship types: 'contains', 'imports', 'calls', 'inherits', 'defines'
  - Node attributes include embeddings for semantic search

**Graph Structure:**
```python
# Single unified graph with typed nodes and edges
G = nx.MultiDiGraph()

# Nodes with type attribute
G.add_node('file_1', type='file', path='src/auth.py', language='python', embedding=[...])
G.add_node('func_1', type='function', name='authenticate', file='file_1', embedding=[...])
G.add_node('class_1', type='class', name='User', file='file_1', embedding=[...])
G.add_node('dep_1', type='dependency', name='requests', version='2.28.0', is_external=True)

# Edges with relationship type
G.add_edge('dir_1', 'file_1', relationship='contains')
G.add_edge('file_1', 'dep_1', relationship='imports')
G.add_edge('func_2', 'func_1', relationship='calls')
G.add_edge('class_2', 'class_1', relationship='inherits')
```

**Benefits:**
- Single file for entire repository graph
- No database installation required
- Fast serialization/deserialization
- Portable and lightweight
- Easy backup and version control
- All relationships in one unified structure

### Starred Messages (Text/File Storage)

**File Format:**
```
Repository: username/repo
URL: https://github.com/username/repo

---

[2024-01-15 10:30:00]
Query: How does authentication work?
Answer: Authentication uses JWT tokens...
Related: auth.py, authenticate()
Tags: authentication, security

---

[2024-01-15 11:45:00]
Query: What is the entry point?
Answer: The main entry point is main.py...
Related: main.py, main()
Tags: architecture, entry-point

---
```

**Storage Location:**
- Local file system: `.repomind/starred/{repo_name}_starred.txt`
- Simple text file format for easy reading and editing
- No database or JSON parsing required

---

## API Design

### REST Endpoints (FastAPI)

```
POST /api/repo
Body: { "githubUrl": "https://github.com/user/repo" }
Response: { "repoId": "uuid", "status": "processing" }
Description: Initiate repository ingestion and graph building

GET /api/repo/{repoId}/status
Response: { 
  "status": "ready" | "processing" | "failed",
  "graphStats": {
    "totalFiles": 1234,
    "parsedFiles": 1200,
    "failedFiles": 34,
    "functionCount": 5678,
    "classCount": 234,
    "dependencyCount": 89
  }
}
Description: Check repository processing status

POST /api/query
Body: { "repoId": "uuid", "query": "How does authentication work?" }
Response: { 
  "answer": "Authentication uses JWT tokens...",
  "sources": ["auth.py", "authenticate()"],
  "relatedNodes": [...]
}
Description: Ask questions about the repository using RAG + Gemini

POST /api/impact
Body: { 
  "repoId": "uuid",
  "target": "authenticate",
  "targetType": "function",
  "maxDepth": 5
}
Response: { 
  "impactLevel": "high",
  "confidence": "moderate",
  "directCallers": ["login", "verify_token"],
  "importingFiles": ["auth.py", "api.py"],
  "affectedNodes": [...],
  "note": "DFS-based impact analysis"
}
Description: Analyze impact of changing a function/class/file

GET /api/graph/{repoId}?type=file|code|deps
Response: { 
  "nodes": [...],
  "edges": [...]
}
Description: Get graph data for visualization

POST /api/star
Body: { 
  "repoId": "uuid",
  "message": "...",
  "query": "...",
  "relatedNodes": [...]
}
Response: { "success": true }
Description: Star an important response (saved to local text file)

GET /api/starred?repoId={repoId}
Response: { "messages": [...] }
Description: Get all starred messages for a repository from text file

POST /api/docs/generate
Body: { "repoId": "uuid" }
Response: { "documentation": "markdown content" }
Description: Generate auto documentation for repository
```

---

## Technology Stack Summary

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Backend Framework | FastAPI (Python) | REST API and backend logic |
| Code Parsing | Python AST + Tree-sitter | Static code analysis |
| Graph Construction | NetworkX | Build and manipulate graphs |
| Graph Storage | Pickle (.pkl files) | Lightweight local storage |
| Traversal Algorithms | NetworkX DFS/BFS | Explanation and impact analysis |
| Embeddings | Sentence Transformers | Semantic search |
| RAG System | Custom pipeline | Retrieve relevant context |
| LLM | Gemini 3 Flash Preview | Generate summaries and Q&A |
| Starred Storage | Text files (.txt) | Local file-based storage |
| Frontend | React + D3.js | UI and graph visualization |
| Deployment | Docker | Containerization |

**Key Differentiators:**
- No database required (Neo4j, PostgreSQL, MongoDB, Redis)
- Lightweight Pickle storage
- NetworkX for all graph operations
- DFS + BFS hybrid traversal
- Gemini 3 Flash Preview for AI generation

---

## Scalability Considerations

### Repository Size Limits
- Target: repos up to ~10k files
- Graph size: up to ~200k nodes
- Pickle files handle this efficiently

### Traversal Limits
- Default depth: 5 hops for DFS impact analysis
- Configurable max: 10 hops
- Timeout: 30 seconds for traversals

### Caching Strategy
- Cache Pickle files in memory after first load
- No persistent cache needed (Pickle is already fast)

### Incremental Updates
- Not supported in v1
- Full re-parse required for updates
- Future: detect changed files via git diff and update graphs incrementally

---

## Security & Privacy

- Only support public repositories (v1)
- No code execution or arbitrary command running
- Clone to isolated temporary directories
- Automatic cleanup after processing
- Rate limiting on API endpoints
- No user authentication in v1 (single-user or demo mode)
- Sanitize all user inputs
- Limit Gemini API token usage per session
- Local storage only (no cloud databases)

---

## Future Enhancements (Post-v1)

- Private repository support with OAuth
- User authentication and multi-user support
- Incremental graph updates (only re-parse changed files)
- IDE plugins (VS Code, JetBrains)
- Multi-repository analysis
- Historical change tracking using git history
- Improved accuracy for dynamic languages
- Support for more languages (Rust, C++, Ruby, Go)
- Real-time collaboration features
- Integration with CI/CD pipelines
- Graph database option (Neo4j) for very large repositories
- Cloud deployment with shared graph storage

---

## Known Limitations (v1)

- Static analysis only; dynamic behavior not captured
- Best-effort parsing; some files may fail
- Impact analysis uses DFS heuristics, not guaranteed
- Single repository per session
- No persistent user accounts
- Limited to ~10k files per repository
- No support for complex monorepos
- Dynamic imports and reflection not fully captured

---

## USP (Unique Selling Points)

1. **Graph-based repository understanding** using NetworkX instead of text summarization
2. **Who-calls-who function call tracing** with DFS/BFS traversal
3. **DFS-based impact analysis** for change propagation
4. **Lightweight local storage** using Pickle (.pkl) - no database required
5. **Gemini + RAG powered** repository Q&A and documentation generation
6. **Local starred responses** feature with simple text file storage
7. **No infrastructure overhead** - works without Neo4j, Redis, or PostgreSQL
