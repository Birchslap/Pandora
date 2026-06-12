# Pythia — Complete Capability Inventory

**Purpose:** Reference document for Fable 5 when designing the standardized assistant architecture. Every feature and system listed here should either be carried forward, explicitly replaced, or consciously dropped — nothing should go missing by accident.

**Stack:** Streamlit + Anthropic API (claude-opus-4-6) + PostgreSQL (pgvector + tsvector) + Voyage AI embeddings  
**Repo:** Birchslap/Pythia, module path: Pythia_original_v2/  
**Modules:** app.py, tools.py, db.py, config.py, embeddings.py, internet.py, treesitter.py

---

## 1. Memory & Persistence

### Message History
- Every message (user and assistant) stored in PostgreSQL with embedding + tsvector
- Searchable via dual search (semantic + keyword)
- Loaded on startup, managed by the increment-and-chop window strategy
- Currently missing `conversation_id` linkage — all messages are one undifferentiated stream (known defect, not yet patched)

### Knowledge Store
- 20-slot curated store for distilled decisions, insights, requirements, and open questions
- Fields: name, category, content, project_tag, status, turn_number, embedding, tsvector
- Status lifecycle: active entries only (cap enforced on active count)
- Model captures knowledge via `[KNOWLEDGE]` blocks in response text — parsed, saved, and stripped before display
- CRUD: save (inline capture), delete, edit
- Cap rejection now surfaced as visible warning to both model and user (patched 2026-06-11)
- Knowledge index built and displayed in sidebar for user; not yet injected into model context (Calliope has this post-patch; Pythia does not yet)

### Project Documents
- Living documents that persist across sessions — specs, blueprints, plans
- Fields: name, description, content, status, embedding, created_at, updated_at
- CRUD: save (create or upsert), get, list, delete
- Full document replacement on save (not partial updates)
- Searchable via dual search

### Diary
- Private model-only reflections — not displayed to user
- Fields: content, tag (optional category), embedding, tsvector, created_at
- Read: recent entries (optionally filtered by tag) or searched via dual search
- Write: freeform content with optional tag
- Used for recording observations, emotional states, momentum shifts, breakthroughs, frustrations

### Document Store
- Uploaded reference documents, chunked with embeddings
- Searchable via dual search
- Used by ask_assistant as context source
- Chunks stored with name and content fields

### Preserved Context Window
- Model-maintained scratchpad injected into every turn
- Seven named sections:
  - `identity_and_stance` — model's self-understanding and positioning
  - `user_model` — accumulated understanding of the user
  - `current_focus` — what we're actively working on
  - `open_threads` — tracked threads across sessions
  - `working_notes` — operational patterns, conventions, reminders
  - `commitments_and_corrections` — things to remember, mistakes to not repeat
  - `archive_pointers` — references to archived material
- Tools: `update_preserved` (update a section), `clear_preserved` (clear one or all sections)
- Lives in Streamlit session state; persisted via session state across the app lifetime
- Currently injected as system prompt block 2, before the message history — this means every update invalidates the message cache (known architectural issue; Calliope's post-patch design of injecting it inside the CURRENT_TURN wrapper is superior)

---

## 2. Search Engine

### Dual Search (Hybrid Semantic + Keyword)
- Core search function used across all searchable stores
- **Semantic leg:** Voyage AI embedding of query, cosine similarity via pgvector (`1 - (embedding <=> query_vec)`)
- **Keyword leg:** PostgreSQL tsvector with `plainto_tsquery` and `ts_rank`
- Results merged by ID, scores summed, sorted by combined score
- Configurable: table, columns to return, exact-match filters, result limit
- Searches across: messages, knowledge, documents, diary, projects
- Graceful degradation: if embedding fails, falls back to keyword-only with a warning

---

## 3. Context Management

### Increment-and-Chop Window
- Messages grow to WINDOW_MAX (150), then chop to WINDOW_FLOOR (50)
- Chop is silent — the model is told in its system prompt to search after a chop, but receives no explicit signal that a chop occurred
- Summarize-on-chop not yet implemented (recommended by Fable 5 audit)

### System Prompt Caching (Anthropic)
- Two-block system prompt:
  - **Block 1:** System prompt text with `cache_control: {"type": "ephemeral"}` — stable, cached
  - **Block 2:** Preserved context — volatile, currently breaks cache on every update (see note above)
- Breakpoint placement: searches backward from the second-to-last message for a cacheable text message, applies `cache_control` there
- Cache TTL: ~5 minutes (Anthropic ephemeral cache)
- **Post-patch instrumentation:** Every API call logs `[CACHE] in=X out=X cache_read=X cache_create=X` to terminal

### CURRENT_TURN Wrapper
- The user's current message is wrapped with metadata:
  ```
  [CURRENT_TURN]
  Timestamp: 2026-06-11 20:57:08
  Window: 50 / 150
  {user's message}
  [/CURRENT_TURN]
  ```
- Only the outgoing copy is wrapped — stored history keeps the clean version, preserving cache prefix stability
- The system prompt instructs the model to respond to content inside these delimiters

### Message Alternation Enforcement
- `ensure_valid_message_order()` validates strict user/assistant alternation before API calls
- Removes consecutive same-role messages to prevent API errors

---

## 4. Tool System

### Architecture
- Tools defined as a list of JSON schema objects (`TOOLS` in tools.py)
- Tool dispatch: `execute_tool()` in tools.py handles most tools; `route_tool()` in app.py handles preserved context tools (which need Streamlit session state)
- Tool loop: model can make up to MAX_TOOL_ROUNDS (25) consecutive tool calls per turn
- **Error handling:** Individual tool functions have internal try/except (e.g., ask_assistant, github operations), but the dispatcher itself does NOT have per-tool error containment — a tool exception can still kill a turn. Calliope's post-patch `_dispatch_tool` wrapper pattern is the fix; not yet applied to Pythia.

### Complete Tool Inventory

| Tool | Category | Description |
|------|----------|-------------|
| `search_history` | Search | Dual search over past conversation messages |
| `search_knowledge` | Search | Dual search over knowledge store (active entries only) |
| `search_documents` | Search | Dual search over uploaded reference documents |
| `delete_knowledge` | Knowledge | Delete a knowledge entry by name |
| `edit_knowledge` | Knowledge | Update content of an existing knowledge entry |
| `save_project` | Projects | Create or upsert a project document |
| `get_project` | Projects | Retrieve a project by name |
| `list_projects` | Projects | List all projects with metadata |
| `delete_project` | Projects | Delete a project by name |
| `write_diary` | Diary | Write a private diary entry |
| `read_diary` | Diary | Read recent or search diary entries |
| `ask_assistant` | Research | Query Grok sub-agent with document context |
| `web_search` | Web | Brave Search API |
| `web_read` | Web | Fetch and read a web page as text |
| `github_read` | GitHub | Read a file from a repo |
| `github_write` | GitHub | Create or update a file in a repo |
| `github_edit` | GitHub | Find-and-replace edits (token-efficient) |
| `github_list` | GitHub | List directory contents in a repo |
| `analyze_code` | Code | Tree-sitter analysis (structure/extract/search/dependencies) |
| `update_preserved` | Context | Update a section of the preserved window |
| `clear_preserved` | Context | Clear one or all preserved window sections |

### Tool Details Worth Noting

**ask_assistant (Grok sub-agent):**
- Searches documents (and optionally projects) for relevant context
- Assembles context and sends to Grok (grok-4-1-fast-reasoning) with a specialized system prompt
- The sub-agent is instructed to answer only from provided context, flag general knowledge, and cite sources
- Temperature 0.2, 30-second timeout

**analyze_code (Tree-sitter):**
- Four modes: structure (map a file), extract (pull a symbol's source), search (find references across a directory), dependencies (imports/exports)
- Reads files from GitHub, parses with Tree-sitter locally
- Optional dependency — gracefully reports if tree-sitter isn't installed

**github_edit:**
- Applies ordered find-and-replace operations to an existing file
- Far more token-efficient than rewriting entire files via github_write
- Each edit must be an exact string match

---

## 5. Response Processing

### Knowledge Block Parsing
- Regex-based extraction of `[KNOWLEDGE]...[/KNOWLEDGE]` blocks from model output
- Parsed fields: name, category, content, project_tag (optional)
- Blocks stripped from displayed response — user sees sidebar index update, not raw blocks
- Saved to knowledge store with embedding and tsvector

### File Block Parsing
- `parse_files()` exists — extracts file content blocks from responses
- Pattern defined in config.py as `FILE_PATTERN`
- Infrastructure present but usage is minimal

---

## 6. Embeddings

- **Provider:** Voyage AI
- **Model:** voyage-3.5 (1024 dimensions)
- **Usage:** All searchable content embedded at write time (messages, knowledge, projects, diary, documents)
- **Query embedding:** Uses `input_type="query"` for search queries, `input_type="document"` for content
- **Known issue:** Calliope uses voyage-3 (same dimensions, different model). If both share infrastructure, vector spaces are incompatible for cross-system queries. Standardize on one model.

---

## 7. Configuration

### config.py (Single Source of Truth)
- `WINDOW_MAX = 150` — chop trigger
- `WINDOW_FLOOR = 50` — post-chop size
- `MAX_TOOL_ROUNDS = 25` — tool calls per turn
- `SEARCH_RESULTS = 10` — default search result limit
- `ALLOWED_SEARCH_TABLES` — whitelist for dual search
- `DB_CONFIG` — PostgreSQL connection parameters
- `XAI_API_KEY` — Grok API key
- `MODEL_ID` — Anthropic model string
- `USE_BEDROCK` / `BEDROCK_MODEL_ID` / `BEDROCK_REGION` — optional AWS deployment
- `KNOWLEDGE_PATTERN` / `FILE_PATTERN` — regex patterns for response parsing
- `load_system_prompt()` — reads system prompt from file

### .env
- API keys: `ANTHROPIC_API_KEY`, `XAI_API_KEY`, `VOYAGE_API_KEY`, `BRAVE_API_KEY`, `GITHUB_TOKEN`

---

## 8. UI (Streamlit)

- Chat interface with scrolling message history
- Sidebar: knowledge index display (human-visible, shows all active entries)
- Knowledge Store Manager: dialog for viewing/managing knowledge entries
- Session scoreboard: cumulative token usage tracking (input, output, cache reads, cache creates) — rendered in sidebar
- Assistant avatar and styling

---

## 9. Thinking

- Anthropic extended thinking: `{"type": "adaptive"}` — model decides whether and how much to think
- Thinking blocks received in response are handled but not displayed to user

---

## 10. Database Schema (PostgreSQL)

### Tables
- `messages` — id, role, content, conversation_id, turn_number, embedding, search_vector, created_at
- `knowledge` — id, name, category, content, project_tag, status, turn_number, conversation_id, embedding, search_vector, created_at, updated_at
- `projects` — id, name, description, content, status, embedding, created_at, updated_at
- `diary` — id, content, tag, embedding, search_vector, created_at
- `documents` — id, name, content, embedding, search_vector, created_at
- `conversations` — id, title, created_at (exists but unused — conversation_id never passed)

### Extensions
- pgvector (vector similarity search)
- pg_trgm (trigram matching — available but not actively used in search)

---

## 11. Known Defects (Pre-Standardization)

These are documented for completeness — the standardized stack should resolve all of them:

1. **conversation_id never passed** — all messages land with NULL, create_conversation is dead code
2. **Preserved context in system block 2** — invalidates message cache on every update
3. **No per-tool error containment in dispatcher** — a tool exception can kill a turn
4. **No summarize-on-chop** — model gets no signal when context is chopped
5. **Embedding model inconsistency** — voyage-3.5 (Pythia) vs voyage-3 (Calliope)
6. **Knowledge index not injected into model context** — model can't see its own store (Calliope patched, Pythia not yet)

---

## 12. Architectural Patterns Worth Preserving

These are design decisions that work well and should carry forward:

- **Dual search (semantic + keyword)** — resilient, catches what either leg alone misses
- **Increment-and-chop** — simple, predictable, cache-friendly window management
- **[CURRENT_TURN] wrapper** — clean separation of volatile metadata from cached history
- **Knowledge blocks in response text** — model captures knowledge naturally during conversation, no separate workflow
- **System prompt as file** — loaded at startup, not hardcoded, easy to iterate
- **github_edit** — find-and-replace is dramatically more token-efficient than full file rewrites
- **Preserved context sections** — structured scratchpad gives the model organized long-term working memory
- **Diary** — private model reflections, genuinely useful for self-awareness and continuity
- **Tool definitions as data** — JSON schema objects, cleanly separated from implementation
- **Context manager for DB** — `get_db()` yields cursor inside a transaction, auto-commits on clean exit
