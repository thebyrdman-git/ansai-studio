# 🎨 ANSAI Studio

**AI-Powered Infrastructure IDE**

> Transform your infrastructure automation with a visual, AI-powered development environment built on ANSAI.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Status](https://img.shields.io/badge/status-in%20development-orange.svg)](https://github.com/thebyrdman-git/ansai-studio)

---

## 🎯 Vision

**ANSAI Studio** brings AI-powered infrastructure intelligence directly into your IDE, making infrastructure management as intuitive as writing code.

### What is ANSAI Studio?

ANSAI Studio is a VS Code extension that provides:

- 🖥️ **Visual Infrastructure Graph** - See your entire infrastructure at a glance
- 🤖 **Local AI Brain** - Intelligent autocomplete and analysis (4GB Ollama model)
- ⚡ **Integrated Playbook Runner** - Execute Ansible from your IDE
- 🔒 **Team Collaboration** - Shared credentials, policy enforcement (Enterprise)
- 💰 **Open Core Model** - Free for individuals, paid for teams

---

## 🏗️ Architecture

ANSAI Studio uses a three-layer architecture:

```
┌─────────────────────────────────────────────────┐
│  Layer 1: IDE Extension (TypeScript)           │
│  • VS Code/VSCodium extension                   │
│  • Infrastructure graph UI                      │
│  • AI autocomplete                              │
└─────────────────────────────────────────────────┘
                ↓ ↑ (HTTP + WebSocket)
┌─────────────────────────────────────────────────┐
│  Layer 2: Local Runtime (Python)                │
│  • HTTP server (127.0.0.1:11434)               │
│  • Token-based security                         │
│  • Ansible execution engine                     │
└─────────────────────────────────────────────────┘
                ↓ ↑ (REST API)
┌─────────────────────────────────────────────────┐
│  Layer 3: AI Brain (Ollama + Cloud)            │
│  • Local: Qwen 2.5 Coder 1.5B (4GB)           │
│  • Cloud: Groq/OpenAI (fallback)               │
└─────────────────────────────────────────────────┘
```

---

## 🚀 Status

**Current Phase:** Foundation Setup (Pre-MVP)

ANSAI Studio is in early development. We're building this based on feedback from the [ANSAI](https://github.com/thebyrdman-git/ansai) community.

### Development Roadmap

- [ ] **Milestone 1: Hello World** (Week 1-2)
  - VS Code extension skeleton
  - Basic Python HTTP server
  - Token-based authentication
  
- [ ] **Milestone 2: Playbook Runner** (Week 3-6)
  - Import Ansible projects
  - Display playbook list
  - Execute playbooks from IDE
  - Stream output to terminal
  
- [ ] **Milestone 3: Infrastructure Graph** (Week 7-12)
  - Parse inventory files
  - Generate visual graph
  - Interactive navigation
  
- [ ] **Milestone 4: AI Brain** (Week 13-16)
  - Ollama integration
  - Background model download
  - AI autocomplete
  - Error analysis

---

## 🛠️ Technology Stack

### Extension (TypeScript)
- VS Code Extension API
- D3.js / Cytoscape.js (graph visualization)
- WebSocket for real-time updates

### Runtime (Python)
- FastAPI (HTTP server)
- ansible-runner (execution engine)
- Token-based authentication

### AI (Ollama + Cloud)
- Local: Qwen 2.5 Coder 1.5B
- Cloud: Groq/OpenAI fallback

---

## 📦 Installation (Coming Soon)

Once MVP is ready, installation will be simple:

```bash
curl -sSL https://raw.githubusercontent.com/thebyrdman-git/ansai-studio/main/install.sh | bash
```

Or install the VS Code extension directly:
```bash
code --install-extension thebyrdman.ansai-studio
```

---

## 🤝 Contributing

ANSAI Studio is built in the open. We welcome contributions!

**How to help:**
- 🐛 Report bugs and issues
- 💡 Suggest features
- 📝 Improve documentation
- 🔨 Submit pull requests

See [CONTRIBUTING.md](CONTRIBUTING.md) (coming soon) for details.

---

## 📋 Project Structure

```
ansai-studio/
├── extension/           # VS Code Extension (TypeScript)
├── runtime/            # Python Backend (FastAPI)
├── ai/                 # AI Integration (Ollama)
├── docs/               # Documentation
├── tests/              # Test suite
└── install.sh          # One-line installer
```

---

## 🔗 Related Projects

- **[ANSAI](https://github.com/thebyrdman-git/ansai)** - The core framework (server-side)
- **[ANSAI Documentation](https://ansai.dev)** - Full documentation

---

## 📄 License

MIT License - See [LICENSE](LICENSE) for details

---

## 💬 Community

- **GitHub Discussions:** https://github.com/thebyrdman-git/ansai-studio/discussions
- **Issues:** https://github.com/thebyrdman-git/ansai-studio/issues
- **Parent Project:** https://github.com/thebyrdman-git/ansai

---

## 🎯 Why ANSAI Studio?

**The Problem:**
- Infrastructure automation is too command-line heavy
- Visualizing complex infrastructure is difficult
- AI tools require cloud APIs (privacy concerns, costs)
- Context switching between editor and terminal kills productivity

**ANSAI Studio's Solution:**
- Everything in your IDE
- Local AI (private, no cloud needed)
- Visual infrastructure graph
- Intelligent autocomplete and analysis
- Zero context switching

---

*Building the future of infrastructure automation, one commit at a time.* 🚀

**ANSAI Studio** • Part of the ANSAI ecosystem

