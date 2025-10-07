# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Retrieval-Augmented Generation (RAG) system for answering questions about course materials. Uses ChromaDB for vector storage, Anthropic's Claude API with tool calling, and a vanilla JavaScript frontend.

**IMPORTANT: This project uses `uv` as the package manager. Always use `uv` commands, never use `pip`.**

## Development Commands

### Setup
```bash
# Install uv package manager (if needed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync

# Create .env file with your API key
echo "ANTHROPIC_API_KEY=your_key_here" > .env
```

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start - ALWAYS use 'uv run', never use pip or python directly
cd backend
uv run uvicorn app:app --reload --port 8000

# To run any Python script in this project
uv run python script_name.py
```

Access points:
- Web UI: `http://localhost:8000`
- API docs: `http://localhost:8000/docs`

### Package Management
```bash
# Add a new dependency - use uv, not pip
uv add package-name

# Remove a dependency
uv remove package-name

# Update dependencies
uv sync
```

## Architecture

### Query Processing Flow

**User Query → FastAPI → RAG System → Claude (with tools) → Vector Search → Response**

1. **Frontend** (script.js): Sends POST to `/api/query` with `{query, session_id}`
2. **FastAPI** (app.py): Routes to `rag_system.query()`
3. **RAG System** (rag_system.py): Orchestrates the entire process
   - Retrieves conversation history from SessionManager
   - Calls AIGenerator with available tools
   - Extracts sources from ToolManager
   - Updates session history
4. **AI Generator** (ai_generator.py): Manages Claude API interaction
   - Claude autonomously decides whether to use the `search_course_content` tool
   - If tool is called: executes search → sends results back to Claude → gets final answer
   - If no tool needed: returns direct answer
5. **Tool Execution** (search_tools.py): When Claude calls the search tool
   - `CourseSearchTool.execute()` receives parameters: query, course_name?, lesson_number?
   - Calls `VectorStore.search()` with filters
   - Formats results with `[Course - Lesson]` headers
   - Tracks sources in `last_sources` for UI display
6. **Vector Store** (vector_store.py): Performs semantic search
   - Resolves partial course names via semantic matching on `course_catalog`
   - Builds metadata filters for course_title and/or lesson_number
   - Queries `course_content` collection with filters
   - Returns `SearchResults` with documents, metadata, distances

### Two-Collection Strategy

**course_catalog** (for semantic course name matching):
- Stores: course titles, instructor names, lesson metadata, links
- Embedding: course title text
- Purpose: Resolve partial course names like "MCP" → "Introduction to MCP"

**course_content** (for actual search):
- Stores: text chunks with metadata (course_title, lesson_number, chunk_index)
- Embedding: chunk content
- Purpose: Semantic search of course material with metadata filtering

### Document Processing

Course documents must follow this format:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [lesson title]
Lesson Link: [url]
[lesson content...]

Lesson 1: [lesson title]
Lesson Link: [url]
[lesson content...]
```

Processing pipeline (document_processor.py):
1. Parse metadata from first 3 lines
2. Split content by "Lesson N:" markers
3. Chunk each lesson using sentence-based chunking (800 char chunks, 100 char overlap)
4. Add context prefix: "Course [title] Lesson [N] content: [chunk]"
5. Create `CourseChunk` objects with metadata

On startup (app.py:88-98):
- Loads all .txt/.pdf/.docx files from `/docs` folder
- Adds to vector store if not already present (checks existing titles)

### Session Management

- Maintains conversation history per session_id
- Stores last 5 Q&A exchanges (10 messages total)
- History passed to Claude for context in follow-up questions

### Tool Calling Pattern

Claude uses Anthropic's tool calling feature:
1. AI receives tool definitions in API call
2. Claude returns `stop_reason: "tool_use"` with tool name and parameters
3. `_handle_tool_execution()` executes the tool via ToolManager
4. Tool results sent back to Claude in new API call
5. Claude generates final answer synthesizing the information

System prompt enforces: **one search per query maximum**

## Configuration (config.py)

Key settings:
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2" (SentenceTransformer)
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: "./chroma_db"

## Key Components

**backend/app.py**: FastAPI server, CORS setup, static file serving, startup document loading

**backend/rag_system.py**: Main orchestrator connecting all components

**backend/vector_store.py**: ChromaDB interface with dual collections and semantic course resolution

**backend/search_tools.py**: Tool abstraction layer for Claude's tool calling

**backend/ai_generator.py**: Claude API client with tool execution loop

**backend/document_processor.py**: Parse course documents and create chunks

**backend/session_manager.py**: Conversation history management

**backend/models.py**: Pydantic models for Course, Lesson, CourseChunk

**frontend/script.js**: Vanilla JS client, handles query submission and response display

**frontend/index.html**: UI with chat interface and course stats sidebar

## Important Patterns

### Adding New Tools

1. Create class inheriting from `Tool` (search_tools.py)
2. Implement `get_tool_definition()` returning Anthropic tool schema
3. Implement `execute(**kwargs)` with tool logic
4. Register in RAGSystem.__init__: `self.tool_manager.register_tool(your_tool)`

### Semantic Course Matching

Users can provide partial course names ("MCP") which are resolved via vector search on course_catalog:
```python
# In vector_store.py
results = self.course_catalog.query(query_texts=["MCP"], n_results=1)
# Returns: "Introduction to MCP"
```

### Source Tracking

Sources flow from search tool → tool manager → RAG system → API → frontend:
1. `CourseSearchTool` stores sources in `self.last_sources`
2. `ToolManager.get_last_sources()` retrieves them
3. RAG system includes in response tuple
4. Frontend displays in collapsible "Sources" section

## Testing Queries

General knowledge: "What is Python?" (no search, direct answer)
Course content: "What is MCP?" (triggers search, returns course content)
Filtered search: "Explain X in course Y" (search with course filter)
- never start the server, I will always do it myself