# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Retrieval-Augmented Generation (RAG) system for querying course materials. Uses ChromaDB for vector storage, Anthropic's Claude API with tool calling, and FastAPI for the backend.

## Development Commands

### Setup
```bash
# Install dependencies
uv sync

# Set up environment variables
# Create .env file with: ANTHROPIC_API_KEY=your_key_here
```

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture

### Core Components

**RAGSystem** (`backend/rag_system.py`) - Main orchestrator that coordinates all components:
- Initializes and manages DocumentProcessor, VectorStore, AIGenerator, SessionManager, and ToolManager
- Handles document ingestion via `add_course_document()` and `add_course_folder()`
- Processes queries through `query()` method which uses AI with tool calling

**VectorStore** (`backend/vector_store.py`) - ChromaDB wrapper with two collections:
- `course_catalog`: Stores course metadata (title, instructor, lessons) for semantic course name matching
- `course_content`: Stores chunked course content with metadata (course_title, lesson_number, chunk_index)
- Key method: `search()` uses semantic search to resolve course names and filter content

**AIGenerator** (`backend/ai_generator.py`) - Anthropic Claude integration:
- Uses tool calling (not RAG retrieval) for search
- Handles tool execution flow: initial request → tool use → tool results → final response
- Model configured in `backend/config.py`: `claude-sonnet-4-20250514`

**ToolManager & CourseSearchTool** (`backend/search_tools.py`) - Tool-based architecture:
- `CourseSearchTool` defines the search interface for Claude to use
- Exposes `search_course_content` tool with parameters: query, course_name (optional), lesson_number (optional)
- Tracks sources from searches for the UI

**DocumentProcessor** (`backend/document_processor.py`) - Parses structured course documents:
- Expected format: Course Title/Link/Instructor on first lines, then "Lesson N: Title" markers
- Creates Course and Lesson objects, chunks content with overlap (configurable)
- Returns Course object and CourseChunk list for vector storage

**SessionManager** (`backend/session_manager.py`) - Conversation history management:
- Tracks conversation context per session_id
- Configurable history length (MAX_HISTORY in config)

### Data Models

**Course, Lesson, CourseChunk** (`backend/models.py`) - Pydantic models defining data structure

### API Layer

**FastAPI app** (`backend/app.py`):
- `POST /api/query`: Process queries, returns answer + sources + session_id
- `GET /api/courses`: Returns course analytics (count, titles)
- On startup: Auto-loads documents from `../docs` folder

### Configuration

**Config** (`backend/config.py`):
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2"
- `CHUNK_SIZE`: 800 chars
- `CHUNK_OVERLAP`: 100 chars
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation turns

## Key Workflows

### Document Ingestion Flow
1. DocumentProcessor reads file and extracts Course + Lessons structure
2. Chunks lesson content with overlap
3. VectorStore adds course metadata to `course_catalog` collection
4. VectorStore adds content chunks to `course_content` collection

### Query Processing Flow
1. User query → RAGSystem.query()
2. AIGenerator receives query with tool definitions
3. Claude decides to use `search_course_content` tool
4. CourseSearchTool executes:
   - If course_name provided, VectorStore semantically resolves it using `course_catalog`
   - Searches `course_content` with filters (course_title, lesson_number)
5. Tool returns formatted results to Claude
6. Claude synthesizes final response
7. Response + sources returned to user

## Important Implementation Details

- The system uses **tool calling**, not traditional RAG retrieval with context injection
- Course name resolution uses semantic search in `course_catalog` collection for fuzzy matching
- Chunks are prefixed with context: "Course [title] Lesson [N] content: [text]"
- Sources are tracked via `CourseSearchTool.last_sources` and reset after each query
- ChromaDB collections persist in `./chroma_db` directory
- Frontend is static files in `../frontend` directory, served by FastAPI
- Always use `uv` to run the server, do not use `pip` directly
- make sure to use `uv` to manage all dependencies.
- use `uv` to run Python files