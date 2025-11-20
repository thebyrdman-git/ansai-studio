# 🎨 ANSAI Studio - Development Handoff Notes

**Status:** Concept phase - Ready for development after ANSAI v1.0 validation  
**Target Start Date:** Q2 2026 (after server-side ANSAI hits 1,000 users)  
**Version:** 2.0  
**Last Updated:** November 20, 2025

---

## 📋 Executive Summary

**ANSAI Studio** is the desktop IDE evolution of ANSAI. It transforms the command-line automation framework into a comprehensive development environment with local AI, visual infrastructure graphs, and team collaboration.

**Current State:** Server-side ANSAI (v1.0) is ready to launch  
**Next Step:** Launch v1.0, validate with community, gather feedback  
**Then:** Build ANSAI Studio based on real user needs

---

## 🎯 Vision Statement

> "Bring AI-powered infrastructure intelligence directly into developers' IDEs, making infrastructure management as intuitive as writing code."

**The Problem We're Solving:**
- Infrastructure automation is still too command-line heavy
- Visualizing complex infrastructure is difficult
- AI tools require cloud APIs (privacy concerns, costs)
- Team collaboration on infrastructure is clunky
- Context switching between editor and terminal kills productivity

**ANSAI Studio's Solution:**
- VS Code extension for infrastructure management
- Local AI (Ollama) with optional cloud fallback
- Real-time infrastructure graph visualization
- Built-in team collaboration and policy enforcement
- Everything in your IDE, no context switching

---

## 📚 Key Documentation

**Essential Reading (In Order):**

1. **[ROADMAP_DESKTOP_IDE.md](ROADMAP_DESKTOP_IDE.md)** - Full vision and technical strategy
2. **[LAUNCH_POSTS.md](docs/LAUNCH_POSTS.md)** - Community positioning
3. **[Current ANSAI docs](https://ansai.dev)** - Understand the foundation

**Architecture References:**
- Security strategy (token-based auth)
- UX strategy ("playable while downloading")
- Sustainability model (open core)
- Three-layer design (IDE → Runtime → AI)

---

## 🏗️ Architecture Overview

### Three-Layer System

```
┌─────────────────────────────────────────────────┐
│  Layer 1: IDE Extension (TypeScript)           │
│  • VS Code/VSCodium extension                   │
│  • Infrastructure graph UI                      │
│  • AI autocomplete                              │
│  • Real-time validation                         │
└─────────────────────────────────────────────────┘
                ↓ ↑ (HTTP + WebSocket)
┌─────────────────────────────────────────────────┐
│  Layer 2: Local Runtime (Python)                │
│  • HTTP server (127.0.0.1:11434)               │
│  • Token-based security                         │
│  • Ansible execution engine                     │
│  • State management                             │
└─────────────────────────────────────────────────┘
                ↓ ↑ (REST API)
┌─────────────────────────────────────────────────┐
│  Layer 3: AI Brain (Ollama + Cloud)            │
│  • Local: Qwen 2.5 Coder 1.5B (4GB)           │
│  • Cloud: Groq/OpenAI (fallback)               │
│  • Model management                             │
└─────────────────────────────────────────────────┘
```

---

## 🔧 Technology Stack

### Layer 1: IDE Extension

**Language:** TypeScript  
**Framework:** VS Code Extension API  
**Key Libraries:**
- `@vscode/extension-api` - VS Code integration
- `d3.js` or `cytoscape.js` - Infrastructure graph visualization
- `monaco-editor` - Code editing (built into VS Code)
- `ws` - WebSocket client for real-time updates

**File Structure:**
```
ansai-studio/
├── extension/
│   ├── src/
│   │   ├── extension.ts          # Entry point
│   │   ├── graph/                # Infrastructure graph
│   │   ├── ai/                   # AI autocomplete
│   │   ├── runtime/              # Runtime communication
│   │   └── ui/                   # Webview panels
│   ├── package.json
│   └── tsconfig.json
```

### Layer 2: Local Runtime

**Language:** Python 3.11+  
**Framework:** FastAPI or Flask  
**Key Libraries:**
- `fastapi` or `flask` - HTTP server
- `ansible-runner` - Execute Ansible playbooks
- `pydantic` - Data validation
- `python-dotenv` - Environment config
- `cryptography` - Token generation/validation

**File Structure:**
```
ansai-studio/
├── runtime/
│   ├── server.py              # HTTP server entry point
│   ├── auth.py                # Token-based auth
│   ├── executor.py            # Ansible execution
│   ├── state.py               # Infrastructure state
│   └── models.py              # Data models
```

### Layer 3: AI Integration

**Local:** Ollama with Qwen 2.5 Coder 1.5B  
**Cloud:** Groq API (primary), OpenAI (fallback)  
**Library:** `ollama-python` or direct HTTP calls

---

## 🔒 Critical Implementation: Security

### Token-Based Authentication

**Problem:** Malicious websites could send `fetch()` requests to `localhost:11434`

**Solution:**

```python
# 1. Generate token on startup (runtime/auth.py)
import uuid
import os

ANSAI_TOKEN = f"ansai-{uuid.uuid4()}"
os.environ['ANSAI_AUTH_TOKEN'] = ANSAI_TOKEN

# Write to .env file for extension to read
with open(os.path.expanduser('~/.ansai/.auth_token'), 'w') as f:
    f.write(ANSAI_TOKEN)
```

```python
# 2. Middleware to validate requests (runtime/server.py)
from fastapi import HTTPException, Request

async def require_token(request: Request):
    auth_header = request.headers.get('Authorization')
    expected = f"Bearer {os.environ['ANSAI_AUTH_TOKEN']}"
    if auth_header != expected:
        raise HTTPException(status_code=401, detail="Unauthorized")
    return True
```

```typescript
// 3. Extension sends token with every request (extension/src/runtime/client.ts)
import * as fs from 'fs';
import * as path from 'path';

const tokenPath = path.join(os.homedir(), '.ansai', '.auth_token');
const token = fs.readFileSync(tokenPath, 'utf-8').trim();

async function callRuntime(endpoint: string, data: any) {
  const response = await fetch(`http://127.0.0.1:11434${endpoint}`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(data)
  });
  return response.json();
}
```

**Security Checklist:**
- [ ] Bind server to `127.0.0.1` ONLY (never `0.0.0.0`)
- [ ] Generate random token on every startup
- [ ] Store token in user home directory (not in project)
- [ ] Validate token on every request
- [ ] Use HTTPS if exposing beyond localhost
- [ ] Implement request rate limiting

---

## 🎨 Critical Implementation: UX

### "Playable While Downloading" Experience

**Problem:** Qwen model is 4GB. Users will uninstall if they wait 20 minutes staring at a spinner.

**Solution: Three-State Activation**

```typescript
// extension/src/ai/downloadManager.ts

enum AIState {
  NOT_INSTALLED = 'not_installed',
  DOWNLOADING = 'downloading',
  READY = 'ready'
}

class AIDownloadManager {
  private state: AIState = AIState.NOT_INSTALLED;
  private progress: number = 0;
  
  async initialize() {
    // Check if Ollama + model exist
    const hasOllama = await this.checkOllama();
    const hasModel = await this.checkModel();
    
    if (hasOllama && hasModel) {
      this.state = AIState.READY;
      this.notifyReady();
    } else {
      this.state = AIState.DOWNLOADING;
      this.startBackgroundDownload();
      this.showStatusBar();
    }
  }
  
  private async startBackgroundDownload() {
    // Non-blocking download
    this.downloadOllama();  // ~200MB
    this.downloadModel();   // ~4GB
    
    // Update progress every 5 seconds
    const interval = setInterval(() => {
      this.updateProgress();
      if (this.progress >= 100) {
        clearInterval(interval);
        this.state = AIState.READY;
        this.notifyReady();
      }
    }, 5000);
  }
  
  private showStatusBar() {
    const item = vscode.window.createStatusBarItem(
      vscode.StatusBarAlignment.Right, 
      100
    );
    item.text = `🧠 AI Brain: Downloading (${this.progress}%)`;
    item.tooltip = 'ANSAI features work now. AI activates when ready.';
    item.show();
  }
  
  private notifyReady() {
    vscode.window.showInformationMessage(
      '🎉 AI Brain Ready! Autocomplete and analysis now active.',
      'Try It'
    ).then(selection => {
      if (selection === 'Try It') {
        this.showAIDemo();
      }
    });
  }
}
```

**UX Checklist:**
- [ ] Core features (playbook editing, graph) work immediately
- [ ] AI features gracefully degrade when not ready
- [ ] Progress indicator is unobtrusive but visible
- [ ] Success notification is celebratory
- [ ] Users never see blocking spinner for > 5 seconds

---

## 📦 Installation Script

### `install.sh` - User Entry Point

**Location:** Root of ansai-studio repo  
**Purpose:** One-line install that gets users to value FAST

```bash
#!/bin/bash
# install-ansai-studio.sh

set -e

ANSAI_HOME="$HOME/.ansai-studio"
RUNTIME_PORT=11434

echo "🚀 Installing ANSAI Studio..."

# 1. Create directories
mkdir -p "$ANSAI_HOME"/{bin,config,data}

# 2. Install Python runtime (FAST - ~50MB)
echo "📦 Installing core runtime..."
curl -L https://github.com/thebyrdman-git/ansai-studio/releases/latest/download/ansai-runtime.tar.gz | \
  tar xz -C "$ANSAI_HOME"

# 3. Install VS Code extension
echo "🛠️  Installing VS Code extension..."
if command -v code &> /dev/null; then
  code --install-extension thebyrdman.ansai-studio
elif command -v codium &> /dev/null; then
  codium --install-extension thebyrdman.ansai-studio
else
  echo "⚠️  VS Code not found. Install extension manually:"
  echo "   https://marketplace.visualstudio.com/items?itemName=thebyrdman.ansai-studio"
fi

# 4. Start background AI download (NON-BLOCKING)
echo "🧠 Starting AI Brain download (runs in background)..."
echo "   You can start using ANSAI Studio now!"
"$ANSAI_HOME/bin/ansai-download-ai" &
DOWNLOAD_PID=$!
echo $DOWNLOAD_PID > "$ANSAI_HOME/.download.pid"

# 5. Start runtime server
echo "🔧 Starting local runtime..."
"$ANSAI_HOME/bin/ansai-runtime" start --port $RUNTIME_PORT --daemon

# 6. Success
echo ""
echo "✅ ANSAI Studio installed!"
echo ""
echo "🎯 Next steps:"
echo "   1. Open VS Code/Codium"
echo "   2. Click ANSAI icon in sidebar"
echo "   3. Import your infrastructure project"
echo ""
echo "💡 AI features activate automatically when download completes."
echo "   Check progress: ansai-studio status"
```

**Install Flow:**
1. Fast core install (~30 seconds)
2. Extension install (if VS Code detected)
3. Background AI download (non-blocking)
4. Runtime server starts
5. User can begin immediately

---

## 🎯 MVP Feature Set

### Phase 1: Minimum Viable Product (2-3 months)

**Goal:** Users can run playbooks from VS Code

**Features:**
- [ ] VS Code extension skeleton
- [ ] Import existing Ansible project
- [ ] File tree view of playbooks
- [ ] Run playbook button
- [ ] Terminal output display
- [ ] Basic error handling

**NOT included in MVP:**
- ❌ Infrastructure graph
- ❌ AI features
- ❌ Team collaboration
- ❌ Visual editing

**Success Metric:** 10 alpha users running playbooks daily

---

### Phase 2: Core Value (3-4 months)

**Goal:** Users see their infrastructure visualized

**Features:**
- [ ] Parse Ansible inventory
- [ ] Generate infrastructure graph
- [ ] Interactive graph navigation
- [ ] Click node to see details
- [ ] Real-time state updates

**Success Metric:** 50 beta users with "this is useful" feedback

---

### Phase 3: AI Integration (4-5 months)

**Goal:** AI makes users faster

**Features:**
- [ ] Ollama integration
- [ ] Background model download
- [ ] AI autocomplete in playbooks
- [ ] Error analysis on failures
- [ ] Natural language queries

**Success Metric:** 70% of users actively use AI features

---

## 📊 Development Milestones

### Milestone 1: Hello World (Week 1-2)
- [ ] Create VS Code extension project
- [ ] Display "Hello ANSAI" in sidebar
- [ ] Create basic Python HTTP server
- [ ] Extension can ping server
- [ ] Token auth works

**Deliverable:** Extension communicates with local server securely

---

### Milestone 2: Playbook Runner (Week 3-6)
- [ ] Import Ansible project
- [ ] Display playbook list
- [ ] Execute playbook via Ansible Runner
- [ ] Stream output to terminal
- [ ] Handle errors

**Deliverable:** Users can run playbooks from VS Code

---

### Milestone 3: Infrastructure Graph (Week 7-12)
- [ ] Parse inventory files
- [ ] Build graph data structure
- [ ] Render graph with D3.js
- [ ] Interactive navigation
- [ ] Node details panel

**Deliverable:** Users see visual infrastructure map

---

### Milestone 4: AI Brain (Week 13-16)
- [ ] Ollama installation
- [ ] Model download manager
- [ ] AI autocomplete
- [ ] Error analysis
- [ ] Query interface

**Deliverable:** AI features work and add value

---

## 🧪 Testing Strategy

### Unit Tests
```typescript
// extension/src/auth/__tests__/tokenValidator.test.ts
describe('Token Validator', () => {
  it('rejects requests without token', async () => {
    const response = await callRuntime('/api/playbooks', {});
    expect(response.status).toBe(401);
  });
  
  it('accepts requests with valid token', async () => {
    const token = readTokenFromFile();
    const response = await callRuntimeWithToken('/api/playbooks', {}, token);
    expect(response.status).toBe(200);
  });
});
```

### Integration Tests
```python
# runtime/tests/test_playbook_execution.py
def test_execute_simple_playbook():
    result = executor.run_playbook('tests/fixtures/hello.yml')
    assert result.status == 'successful'
    assert 'Hello World' in result.stdout
```

### E2E Tests
```typescript
// extension/tests/e2e/complete-flow.test.ts
test('complete user flow', async () => {
  // 1. Install extension
  // 2. Import project
  // 3. Run playbook
  // 4. Verify output
  // 5. View graph
});
```

---

## 🚧 Known Challenges & Solutions

### Challenge 1: Ollama Model Size (4GB)

**Problem:** Large download, users will quit  
**Solution:** "Playable while downloading" UX (see above)  
**Backup Plan:** Offer cloud-only mode (no local AI)

---

### Challenge 2: Cross-Platform Support

**Problem:** Windows, Mac, Linux all behave differently  
**Solution:** 
- Phase 1: Mac/Linux only (80% of DevOps users)
- Phase 2: Add Windows support
- Use Python for runtime (cross-platform)
- VS Code handles UI differences

---

### Challenge 3: State Synchronization

**Problem:** Keep extension UI in sync with infrastructure changes  
**Solution:**
- WebSocket connection for real-time updates
- Poll state every 30 seconds as fallback
- Optimistic UI updates (assume success)
- Reconcile on WebSocket reconnect

---

### Challenge 4: Team Collaboration

**Problem:** Multiple developers editing same infrastructure  
**Solution:**
- Phase 1: Single user only
- Phase 2: Git-based collaboration
- Phase 3: Live collaboration (future)

---

## 💰 Monetization (Enterprise Tier)

### Free Tier (Community Edition)

**Always Free:**
- Local runtime
- Local AI (Ollama)
- Single user
- Read-only infrastructure graph
- Basic features

---

### Paid Tier ($49/month - Team)

**What's added:**
- Team Wallet (shared encrypted credentials)
- Policy Enforcer (prevent dangerous ops)
- Cloud Sync Dashboard
- Priority support
- Cloud AI models (OpenAI, Claude)

---

### Enterprise ($199/month)

**What's added:**
- SSO integration
- Audit logs
- Compliance reports
- Custom integrations
- Dedicated support

---

## 📝 Open Questions (Needs Decisions)

### Technical Decisions

1. **Python Web Framework:**
   - Option A: FastAPI (modern, async, OpenAPI)
   - Option B: Flask (simple, stable, proven)
   - **Recommendation:** FastAPI

2. **Graph Library:**
   - Option A: D3.js (flexible, steep learning curve)
   - Option B: Cytoscape.js (graph-specific, easier)
   - **Recommendation:** Cytoscape.js

3. **State Management:**
   - Option A: SQLite (simple, local)
   - Option B: JSON files (simpler, no DB)
   - **Recommendation:** JSON files for MVP

4. **Real-time Updates:**
   - Option A: WebSocket (bi-directional, complex)
   - Option B: Server-Sent Events (simple, one-way)
   - **Recommendation:** WebSocket

---

### Product Decisions

1. **Launch Timing:**
   - After ANSAI v1.0 hits 1,000 users? 500? 100?
   - **Recommendation:** 1,000 users OR 3 months, whichever comes first

2. **MVP Scope:**
   - Include AI in MVP or wait for Phase 2?
   - **Recommendation:** No AI in MVP, focus on playbook runner

3. **Platform Priority:**
   - Mac first? Linux first? Both?
   - **Recommendation:** Mac + Linux together (Windows later)

4. **Naming:**
   - "ANSAI Studio" confirmed?
   - **Recommendation:** Yes, locked in

---

## 📚 Resources & References

### Technical Learning

**VS Code Extension Development:**
- https://code.visualstudio.com/api/get-started/your-first-extension
- https://github.com/microsoft/vscode-extension-samples

**Ansible Runner (Python):**
- https://ansible-runner.readthedocs.io/

**Ollama:**
- https://ollama.ai/docs

**Qwen 2.5 Coder:**
- https://ollama.ai/library/qwen2.5-coder

---

### Inspiration (Similar Tools)

- **Terraform Plugin for VS Code** - Infrastructure visualization
- **Kubernetes Extension** - Graph-based UI
- **GitHub Copilot** - AI autocomplete UX
- **Docker Desktop** - Local runtime management

---

## 🎯 Success Metrics

### Adoption Metrics
- 10,000+ downloads in Month 1
- 1,000+ daily active users by Month 3
- 40% week-1 retention
- 25% month-1 retention

### Engagement Metrics
- 50%+ users run playbooks daily
- 70%+ users use infrastructure graph
- 60%+ users activate AI features
- 30+ min average daily usage

### Business Metrics
- 2% free → paid conversion
- $5K+ MRR by Month 6
- <5% monthly churn
- 40+ NPS score

---

## 🚦 Go/No-Go Criteria

### Before Starting Development

**Must Have:**
- ✅ ANSAI v1.0 launched and stable
- ✅ 1,000+ active users OR strong community demand
- ✅ 100+ users say "I'd use a desktop version"
- ✅ Clear product-market fit validated
- ✅ Funding or time commitment secured

**Nice to Have:**
- Community contributors interested
- VS Code Marketplace approval confirmed
- Beta testing group committed

---

## 👥 Team Needs

### Ideal Team (If Funded)

**Full-Time:**
- 1x Senior TypeScript Dev (Extension)
- 1x Senior Python Dev (Runtime)
- 1x UX/UI Designer (Graph + Flows)

**Part-Time:**
- 1x DevRel (Community)
- 1x Technical Writer (Docs)

**Budget:** ~$300K/year (3 FT salaries + overhead)

---

### Solo Developer Path

**If building alone:**

**Phase 1 (Months 1-2):** Foundation
- Focus: Basic extension + runtime
- Scope: Playbook runner only
- Goal: 10 alpha users

**Phase 2 (Months 3-4):** Core Value
- Focus: Infrastructure graph
- Scope: Visualization only
- Goal: 50 beta users

**Phase 3 (Months 5-6):** AI
- Focus: Ollama integration
- Scope: Autocomplete + analysis
- Goal: 100+ users, 70% using AI

**Realistic Timeline:** 6-9 months solo to v1.0

---

## 📁 Starter Project Structure

```
ansai-studio/
├── README.md
├── LICENSE (MIT)
├── .gitignore
│
├── extension/                 # VS Code Extension
│   ├── src/
│   │   ├── extension.ts       # Entry point
│   │   ├── views/             # UI components
│   │   ├── runtime/           # Runtime communication
│   │   ├── graph/             # Infrastructure graph
│   │   └── ai/                # AI features
│   ├── package.json
│   ├── tsconfig.json
│   └── README.md
│
├── runtime/                   # Python Backend
│   ├── server.py              # FastAPI server
│   ├── auth.py                # Token auth
│   ├── executor.py            # Ansible runner
│   ├── state.py               # State management
│   ├── models.py              # Data models
│   ├── requirements.txt
│   └── README.md
│
├── ai/                        # AI Integration
│   ├── ollama.py              # Ollama client
│   ├── download.py            # Model downloader
│   ├── autocomplete.py        # AI autocomplete
│   └── analysis.py            # Error analysis
│
├── install.sh                 # One-line installer
├── docs/                      # Documentation
└── tests/                     # Test suite
```

---

## 🎬 Next Steps

### Immediate (After ANSAI v1.0 Launch)

1. **Validate Demand**
   - Survey ANSAI users
   - Run polls in Discussions
   - Gauge interest level

2. **Build Community**
   - Recruit beta testers
   - Find contributors
   - Create ANSAI Studio waiting list

3. **Refine Vision**
   - Incorporate user feedback
   - Prioritize features
   - Adjust roadmap

---

### When Ready to Start

1. **Set up repos**
   - `ansai-studio` (main)
   - `ansai-studio-extension` (VS Code)
   - `ansai-studio-runtime` (Python)

2. **Build MVP**
   - Week 1-2: Hello World
   - Week 3-6: Playbook runner
   - Week 7: Alpha release

3. **Iterate**
   - Weekly releases
   - Gather feedback
   - Adjust course

---

## 💬 Contact & Support

**Questions about this handoff?**
- GitHub Discussions: https://github.com/thebyrdman-git/ansai/discussions
- Label: "ansai-studio"

**Want to contribute?**
- Comment on GitHub issue: "ANSAI Studio Development"
- Join waiting list (coming soon)

---

## 📌 Final Notes

### What NOT to Do

❌ **Don't start building before v1.0 validation**
- You'll waste time building the wrong thing
- Community feedback is invaluable
- Market validation prevents expensive pivots

❌ **Don't try to build everything at once**
- Start with playbook runner only
- Add graph second
- AI is phase 3, not phase 1

❌ **Don't ignore security**
- Token auth is critical
- Localhost-only binding is non-negotiable
- Test security first, features second

---

### What TO Do

✅ **Launch ANSAI v1.0 first**
- Get users
- Get feedback
- Validate AI + infrastructure combo

✅ **Start small, iterate fast**
- MVP is playbook runner
- Ship weekly
- Learn constantly

✅ **Build with the community**
- Open source from day 1
- Accept contributions
- Listen to feedback

---

**This handoff document will evolve.** Update it as you learn more from the ANSAI v1.0 community.

**Good luck building ANSAI Studio!** 🚀

---

*Document maintained by: ANSAI Core Team*  
*Last updated: November 20, 2025*  
*Next review: After ANSAI v1.0 hits 1,000 users*

