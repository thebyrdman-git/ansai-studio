# Red Hat Template-Agent Insights for ANSAI Studio

**Source:** https://github.com/redhat-data-and-ai/template-agent  
**Analysis Date:** November 20, 2025  
**Purpose:** Identify production-ready patterns and best practices to incorporate into ANSAI Studio

---

## 🎯 Executive Summary

The Red Hat template-agent is a **production-ready FastAPI template** for building AI agents with enterprise features. It provides excellent patterns for:

1. **FastAPI architecture** with proper separation of concerns
2. **Settings management** using Pydantic BaseSettings
3. **Streaming APIs** with Server-Sent Events (SSE)
4. **Database integration** with PostgreSQL checkpointing
5. **Enterprise features** like health checks, logging, tracing
6. **Security patterns** including SSL/TLS and authentication

These patterns directly apply to ANSAI Studio's Runtime layer (Python backend).

---

## 🏗️ Architecture Comparison

### Red Hat Template-Agent

```
Client (UI/API)
    ↓
FastAPI Application (api.py)
    ↓
Routes (health, stream, history, threads, feedback)
    ↓
Core Layer (agent, agent_utils, prompt)
    ↓
Data Layer (PostgreSQL, Langfuse)
    ↓
External Services (Google AI, SSO)
```

### ANSAI Studio (Our Design)

```
VS Code Extension (TypeScript)
    ↓
FastAPI Runtime (runtime/server.py)
    ↓
Routes (playbooks, inventory, graph, ai)
    ↓
Core Layer (executor, state, auth)
    ↓
Data Layer (JSON files / SQLite)
    ↓
External Services (Ollama, Groq, Ansible)
```

**Insight:** Our architecture aligns well! We should adopt their patterns for routes, settings, and lifecycle management.

---

## 📚 Key Patterns to Adopt

### 1. Settings Management (Pydantic BaseSettings)

**What They Do:**

```python
# template_agent/src/settings.py
from pydantic import Field
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Server Configuration
    AGENT_HOST: str = Field(default="0.0.0.0")
    AGENT_PORT: int = Field(default=8081)
    PYTHON_LOG_LEVEL: str = Field(default="INFO")
    
    # Database Configuration
    POSTGRES_USER: str = Field(default="pgvector")
    POSTGRES_PASSWORD: str = Field(default="pgvector")
    
    # Computed properties
    @property
    def database_uri(self) -> str:
        return f"postgresql://{self.POSTGRES_USER}:{self.POSTGRES_PASSWORD}@..."

settings = Settings()
```

**How ANSAI Studio Should Use This:**

```python
# ansai-studio/runtime/settings.py
from pydantic import Field
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Server Configuration
    ANSAI_HOST: str = Field(default="127.0.0.1")  # localhost only!
    ANSAI_PORT: int = Field(default=11434)
    PYTHON_LOG_LEVEL: str = Field(default="INFO")
    
    # Security
    ANSAI_AUTH_TOKEN: str = Field(default="")  # Generated on startup
    
    # Ansible Configuration
    ANSIBLE_PROJECT_PATH: str = Field(default="")
    ANSIBLE_VAULT_PASSWORD: str = Field(default="")
    
    # AI Configuration
    OLLAMA_HOST: str = Field(default="http://localhost:11434")
    GROQ_API_KEY: str = Field(default="")
    OPENAI_API_KEY: str = Field(default="")
    
    # State Management
    USE_SQLITE: bool = Field(default=False)  # False = JSON files
    STATE_DIR: str = Field(default="~/.ansai-studio/state")

settings = Settings()
```

**Benefits:**
- ✅ Type-safe configuration
- ✅ Environment variable validation
- ✅ Default values
- ✅ Easy testing (override settings)

---

### 2. Lifecycle Management (Async Context Managers)

**What They Do:**

```python
# template_agent/src/api.py
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """Manage application lifecycle - startup and shutdown."""
    # Startup
    logger.info("Starting Template Agent")
    await initialize_database()
    
    yield  # Application runs here
    
    # Shutdown
    logger.info("Shutting down Template Agent")
    # Cleanup resources

app = FastAPI(lifespan=lifespan)
```

**How ANSAI Studio Should Use This:**

```python
# ansai-studio/runtime/server.py
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """Manage ANSAI Studio runtime lifecycle."""
    # Startup
    logger.info("Starting ANSAI Studio Runtime")
    
    # Generate auth token
    auth_token = generate_auth_token()
    save_token_to_file(auth_token)
    
    # Initialize state directory
    initialize_state_dir(settings.STATE_DIR)
    
    # Check for Ollama
    ollama_available = await check_ollama()
    if not ollama_available:
        logger.warning("Ollama not available - AI features disabled")
    
    yield  # Application runs
    
    # Shutdown
    logger.info("Shutting down ANSAI Studio Runtime")
    # Close any open connections

app = FastAPI(lifespan=lifespan)
```

**Benefits:**
- ✅ Proper resource initialization
- ✅ Graceful shutdown
- ✅ Clean separation of startup/shutdown logic

---

### 3. Route Organization

**What They Do:**

```
template_agent/src/routes/
├── health.py      # Health checks
├── stream.py      # Streaming chat
├── history.py     # Chat history
├── threads.py     # Thread management
└── feedback.py    # Feedback recording
```

Each route file has:
- Router definition
- Pydantic request/response models
- Route handlers
- Documentation

**How ANSAI Studio Should Use This:**

```
ansai-studio/runtime/routes/
├── health.py        # Health checks
├── playbooks.py     # List/execute playbooks
├── inventory.py     # Infrastructure inventory
├── graph.py         # Infrastructure graph data
├── ai.py            # AI autocomplete/analysis
├── auth.py          # Token validation
└── state.py         # State management
```

**Example:**

```python
# ansai-studio/runtime/routes/playbooks.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter(prefix="/api/playbooks", tags=["playbooks"])

class PlaybookExecuteRequest(BaseModel):
    playbook_path: str
    extra_vars: dict = {}
    check_mode: bool = False

class PlaybookExecuteResponse(BaseModel):
    execution_id: str
    status: str
    message: str

@router.post("/execute", response_model=PlaybookExecuteResponse)
async def execute_playbook(request: PlaybookExecuteRequest):
    """Execute an Ansible playbook."""
    # Implementation here
    pass

@router.get("/list")
async def list_playbooks():
    """List available playbooks."""
    # Implementation here
    pass
```

**Benefits:**
- ✅ Clear separation of concerns
- ✅ Easy to test individual routes
- ✅ Auto-generated OpenAPI docs

---

### 4. Middleware for Logging

**What They Do:**

```python
class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """Middleware to log all incoming requests and outgoing responses."""
    
    async def dispatch(self, request: Request, call_next: Callable):
        start_time = time.time()
        
        # Log request
        logger.info("incoming_request", 
                   method=request.method,
                   path=request.url.path)
        
        response = await call_next(request)
        
        # Log response
        duration_ms = (time.time() - start_time) * 1000
        logger.info("outgoing_response",
                   status_code=response.status_code,
                   duration_ms=round(duration_ms, 2))
        
        return response

app.add_middleware(RequestLoggingMiddleware)
```

**How ANSAI Studio Should Use This:**

```python
# ansai-studio/runtime/middleware/auth.py
class TokenAuthMiddleware(BaseHTTPMiddleware):
    """Middleware to validate auth token on every request."""
    
    async def dispatch(self, request: Request, call_next: Callable):
        # Skip auth for health check
        if request.url.path == "/health":
            return await call_next(request)
        
        # Validate token
        auth_header = request.headers.get("Authorization")
        expected_token = f"Bearer {settings.ANSAI_AUTH_TOKEN}"
        
        if auth_header != expected_token:
            return JSONResponse(
                status_code=401,
                content={"error": "Unauthorized"}
            )
        
        return await call_next(request)

app.add_middleware(TokenAuthMiddleware)
```

**Benefits:**
- ✅ Security applied to all routes automatically
- ✅ Centralized logging
- ✅ Performance monitoring

---

### 5. Streaming API with Server-Sent Events (SSE)

**What They Do:**

```python
# template_agent/src/routes/stream.py
@router.post("/v1/stream")
async def stream_chat(request: StreamRequest):
    """Stream chat responses using Server-Sent Events."""
    
    async def generate_events():
        # Stream tokens
        yield {"type": "token", "content": "Hello"}
        yield {"type": "token", "content": " world"}
        
        # Final message
        yield {"type": "message", "content": {"type": "ai", "content": "Hello world"}}
        yield "[DONE]"
    
    return EventSourceResponse(generate_events())
```

**How ANSAI Studio Should Use This:**

```python
# ansai-studio/runtime/routes/playbooks.py
@router.post("/execute/stream")
async def execute_playbook_stream(request: PlaybookExecuteRequest):
    """Execute playbook and stream output in real-time."""
    
    async def stream_output():
        async for event in ansible_runner.run_streaming(request.playbook_path):
            if event.type == "task_start":
                yield {"type": "task_start", "task": event.task_name}
            elif event.type == "stdout":
                yield {"type": "stdout", "line": event.line}
            elif event.type == "task_complete":
                yield {"type": "task_complete", "task": event.task_name, "status": event.status}
        
        yield "[DONE]"
    
    return EventSourceResponse(stream_output())
```

**Benefits:**
- ✅ Real-time feedback to extension
- ✅ No need for polling
- ✅ Browser-native (EventSource API)

---

### 6. Exception Handling

**What They Do:**

```python
# template_agent/src/core/exceptions/exceptions.py
from enum import Enum

class AppExceptionCode(str, Enum):
    CONFIGURATION_VALIDATION_ERROR = "CONFIGURATION_VALIDATION_ERROR"
    CONFIGURATION_INITIALIZATION_ERROR = "CONFIGURATION_INITIALIZATION_ERROR"
    DATABASE_CONNECTION_ERROR = "DATABASE_CONNECTION_ERROR"

class AppException(Exception):
    """Custom application exception with error codes."""
    
    def __init__(self, message: str, code: AppExceptionCode):
        self.message = message
        self.code = code
        super().__init__(self.message)

# Usage:
raise AppException(
    "Database initialization failed",
    AppExceptionCode.DATABASE_CONNECTION_ERROR
)
```

**How ANSAI Studio Should Use This:**

```python
# ansai-studio/runtime/exceptions.py
class AnsaiExceptionCode(str, Enum):
    AUTH_TOKEN_INVALID = "AUTH_TOKEN_INVALID"
    PLAYBOOK_NOT_FOUND = "PLAYBOOK_NOT_FOUND"
    PLAYBOOK_EXECUTION_FAILED = "PLAYBOOK_EXECUTION_FAILED"
    ANSIBLE_NOT_INSTALLED = "ANSIBLE_NOT_INSTALLED"
    OLLAMA_NOT_AVAILABLE = "OLLAMA_NOT_AVAILABLE"

class AnsaiException(Exception):
    def __init__(self, message: str, code: AnsaiExceptionCode):
        self.message = message
        self.code = code
        super().__init__(self.message)
```

**Benefits:**
- ✅ Consistent error handling
- ✅ Error codes for client parsing
- ✅ Better debugging

---

### 7. Testing Structure

**What They Have:**

```
tests/
├── test_agent_utils.py
├── test_database_init.py
├── test_exceptions.py
├── test_feedback.py
├── test_health.py
├── test_prompt.py
├── test_schema.py
└── test_settings.py
```

Each test file:
- Tests one module
- Uses pytest fixtures
- Includes async tests
- Has good coverage

**How ANSAI Studio Should Organize Tests:**

```
tests/
├── unit/
│   ├── test_auth.py
│   ├── test_settings.py
│   ├── test_exceptions.py
│   └── test_state.py
├── integration/
│   ├── test_ansible_runner.py
│   ├── test_ollama_client.py
│   └── test_api_routes.py
└── e2e/
    └── test_complete_flow.py
```

---

## 🔒 Security Patterns to Adopt

### 1. Localhost Binding (Already in Our Design)

```python
# They use 0.0.0.0 (accessible from network)
AGENT_HOST: str = Field(default="0.0.0.0")

# We should use 127.0.0.1 (localhost only)
ANSAI_HOST: str = Field(default="127.0.0.1")
```

### 2. SSL/TLS Support

```python
# runtime/server.py
if settings.ANSAI_SSL_CERTFILE and settings.ANSAI_SSL_KEYFILE:
    uvicorn.run(
        app,
        host=settings.ANSAI_HOST,
        port=settings.ANSAI_PORT,
        ssl_keyfile=settings.ANSAI_SSL_KEYFILE,
        ssl_certfile=settings.ANSAI_SSL_CERTFILE
    )
```

### 3. Configuration Validation

```python
def validate_config(settings: Settings) -> None:
    """Validate configuration on startup."""
    if not (1024 <= settings.ANSAI_PORT <= 65535):
        raise AnsaiException(
            f"Port must be between 1024 and 65535",
            AnsaiExceptionCode.CONFIGURATION_ERROR
        )
    
    if settings.ANSAI_HOST != "127.0.0.1":
        logger.warning("ANSAI_HOST is not 127.0.0.1 - security risk!")
```

---

## 📊 Database Strategy Comparison

### Red Hat Template-Agent

- **Primary:** PostgreSQL with async support
- **Checkpointing:** LangGraph AsyncPostgresSaver
- **State:** Persisted conversation history
- **Fallback:** In-memory saver for dev

### ANSAI Studio (Recommendation)

- **Primary:** JSON files (simpler, no DB server)
- **State:** `~/.ansai-studio/state/`
  - `inventory.json`
  - `playbooks.json`
  - `execution_history.json`
  - `graph.json`
- **Optional:** SQLite for complex queries
- **Rationale:** 
  - Simpler deployment (no DB to manage)
  - Good enough for single user
  - Can upgrade to SQLite/Postgres later

**Implementation:**

```python
# ansai-studio/runtime/state.py
import json
from pathlib import Path
from typing import Any, Dict

class StateManager:
    """Manage ANSAI Studio state using JSON files."""
    
    def __init__(self, state_dir: str):
        self.state_dir = Path(state_dir).expanduser()
        self.state_dir.mkdir(parents=True, exist_ok=True)
    
    def save(self, key: str, data: Dict[str, Any]):
        """Save state to JSON file."""
        file_path = self.state_dir / f"{key}.json"
        with open(file_path, 'w') as f:
            json.dump(data, f, indent=2)
    
    def load(self, key: str) -> Dict[str, Any]:
        """Load state from JSON file."""
        file_path = self.state_dir / f"{key}.json"
        if not file_path.exists():
            return {}
        with open(file_path, 'r') as f:
            return json.load(f)
```

---

## 🎯 Recommended Implementation Order

### Phase 1: Foundation (Week 1-2)

✅ **From Template-Agent:**
1. Adopt Pydantic BaseSettings pattern
2. Implement lifecycle management (@asynccontextmanager)
3. Create route structure (health, playbooks, inventory)
4. Add request logging middleware
5. Implement exception handling with error codes

### Phase 2: Core Features (Week 3-6)

✅ **From Template-Agent:**
1. Streaming API for playbook execution (SSE)
2. State management with JSON files
3. Health check endpoint
4. Comprehensive logging

### Phase 3: Advanced Features (Week 7+)

⚠️ **Adapt from Template-Agent:**
1. Optional PostgreSQL support for teams
2. Tracing/observability (like their Langfuse integration)
3. Advanced error handling

---

## 📝 Action Items for ANSAI Studio

### Immediate (This Week)

- [x] Create this analysis document
- [ ] Copy settings.py pattern to runtime/
- [ ] Implement lifecycle management
- [ ] Create route structure
- [ ] Add middleware (auth + logging)

### Short Term (Next 2 Weeks)

- [ ] Implement streaming API for playbook execution
- [ ] Add comprehensive exception handling
- [ ] Create state management with JSON
- [ ] Write unit tests following their structure

### Medium Term (Next Month)

- [ ] Add health checks and monitoring
- [ ] Implement optional SQLite support
- [ ] Add tracing/observability
- [ ] Performance optimization

---

## 💡 Key Takeaways

### What to Adopt Directly

1. ✅ **Pydantic BaseSettings** - Exact pattern, proven
2. ✅ **Route structure** - Clean separation, scalable
3. ✅ **Middleware pattern** - Auth + logging centralized
4. ✅ **Lifecycle management** - Proper startup/shutdown
5. ✅ **Exception handling** - Error codes + consistency

### What to Adapt

1. ⚠️ **Database** - JSON files instead of PostgreSQL (for now)
2. ⚠️ **Streaming** - SSE for playbook output (not chat)
3. ⚠️ **Authentication** - Token-based (already in our design)

### What to Skip

1. ❌ **Langfuse integration** - Not needed for MVP
2. ❌ **Google AI** - We use Ollama/Groq
3. ❌ **MCP client** - Different architecture

---

## 🔗 Resources

- **Template-Agent Repo:** https://github.com/redhat-data-and-ai/template-agent
- **FastAPI Docs:** https://fastapi.tiangolo.com/
- **Pydantic Settings:** https://docs.pydantic.dev/latest/concepts/pydantic_settings/
- **SSE in FastAPI:** https://github.com/sysid/sse-starlette

---

## 📌 Conclusion

The Red Hat template-agent provides **production-ready patterns** that directly apply to ANSAI Studio's runtime layer. By adopting their:

- Settings management
- Route organization
- Lifecycle management
- Middleware patterns
- Testing structure

We can build a more **robust, maintainable, and enterprise-ready** runtime server.

**Next step:** Implement these patterns in our runtime/ directory starting with settings.py and basic route structure.

---

*Document created: November 20, 2025*  
*Author: ANSAI Studio Development Team*  
*Status: Ready for implementation*

