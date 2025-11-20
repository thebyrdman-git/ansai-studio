# ANSAI Studio Architecture

## Overview

ANSAI Studio is built on a three-layer architecture designed for security, performance, and extensibility.

## Layer 1: IDE Extension (TypeScript)

**Location:** `/extension`

**Purpose:** User interface and interaction layer

**Components:**
- VS Code Extension API integration
- Infrastructure graph visualization (D3.js/Cytoscape.js)
- AI autocomplete interface
- Real-time state updates (WebSocket)
- Command palette integration

**Key Files:**
- `extension.ts` - Entry point
- `views/` - UI components and webviews
- `runtime/` - Communication with backend
- `graph/` - Infrastructure visualization
- `ai/` - AI feature integration

## Layer 2: Local Runtime (Python)

**Location:** `/runtime`

**Purpose:** Secure execution engine and state management

**Components:**
- FastAPI HTTP server (127.0.0.1:11434)
- Token-based authentication
- Ansible playbook execution (ansible-runner)
- Infrastructure state management
- WebSocket server for real-time updates

**Key Files:**
- `server.py` - FastAPI application
- `auth.py` - Token generation and validation
- `executor.py` - Ansible playbook runner
- `state.py` - State management
- `models.py` - Pydantic data models

**Security:**
- Binds to localhost only (never 0.0.0.0)
- Generates random token on startup
- Validates token on every request
- Token stored in `~/.ansai-studio/.auth_token`

## Layer 3: AI Integration

**Location:** `/ai`

**Purpose:** Intelligent analysis and assistance

**Components:**
- Ollama client (local AI)
- Model download manager
- Cloud AI fallback (Groq/OpenAI)
- Autocomplete engine
- Error analysis

**Key Files:**
- `ollama.py` - Local Ollama client
- `download.py` - Model download with progress
- `autocomplete.py` - AI-powered autocomplete
- `analysis.py` - Error root cause analysis

**Models:**
- Primary: Qwen 2.5 Coder 1.5B (4GB)
- Fallback: Cloud providers

## Communication Flow

```
User Action in VS Code
    ↓
Extension captures event
    ↓
HTTP Request to Runtime (with auth token)
    ↓
Runtime validates token
    ↓
Executor runs Ansible playbook
    ↓
AI analyzes output (if needed)
    ↓
WebSocket pushes updates to Extension
    ↓
UI updates in real-time
```

## Data Flow

### Infrastructure Graph
```
Ansible Inventory → Parser → Graph Data Model → D3.js Visualization
```

### AI Autocomplete
```
User types → Extension → Runtime → Ollama → Suggestions → Extension → UI
```

### Playbook Execution
```
User clicks Run → Extension → Runtime → ansible-runner → Output Stream → WebSocket → Terminal
```

## Security Architecture

### Token-Based Authentication

1. **Startup:** Runtime generates random UUID token
2. **Storage:** Token written to `~/.ansai-studio/.auth_token`
3. **Extension:** Reads token from file
4. **Request:** Every HTTP request includes `Authorization: Bearer <token>`
5. **Validation:** Runtime middleware validates token

### Localhost Binding

- Runtime server binds to `127.0.0.1` only
- Prevents external network access
- Malicious websites cannot connect

### Token Rotation

- New token generated on every runtime restart
- Old tokens immediately invalid
- Automatic synchronization with extension

## State Management

### Infrastructure State

Stored in JSON format:
```
~/.ansai-studio/state/
├── inventory.json       # Parsed inventory
├── playbooks.json       # Available playbooks
├── execution_history.json  # Past runs
└── graph.json          # Graph structure
```

### Execution History

Tracks:
- Playbook runs
- Execution time
- Success/failure
- AI analysis results
- Error patterns

## Extensibility

### Plugin System (Future)

- Custom visualizations
- Additional AI providers
- Custom healing strategies
- Third-party integrations

### API Design

REST API follows OpenAPI 3.0 spec:
- `/api/playbooks` - List/execute playbooks
- `/api/inventory` - Infrastructure data
- `/api/graph` - Graph structure
- `/api/ai/complete` - AI autocomplete
- `/api/ai/analyze` - Error analysis

WebSocket endpoints:
- `/ws/execution` - Playbook output stream
- `/ws/state` - Real-time state updates

## Performance Considerations

### UI Responsiveness

- Non-blocking AI downloads
- Optimistic UI updates
- Progressive graph rendering
- Debounced autocomplete

### Resource Management

- Lazy load AI model
- Cache parsed inventory
- Stream large outputs
- Limit concurrent executions

## Development Workflow

```
1. Extension (TypeScript) → Compile with tsc
2. Runtime (Python) → Run with uvicorn
3. AI (Python) → Manage with download script
4. Extension tests → VS Code Test Runner
5. Runtime tests → pytest
```

## Deployment

### Local Development
```bash
# Terminal 1: Runtime
cd runtime && python server.py

# Terminal 2: Extension
cd extension && npm run watch

# VS Code: Press F5 to launch Extension Development Host
```

### Production Package
```bash
./build.sh
# Creates:
# - ansai-studio.vsix (extension)
# - ansai-runtime.tar.gz (runtime)
# - install.sh (installer)
```

---

**Last Updated:** November 20, 2025

