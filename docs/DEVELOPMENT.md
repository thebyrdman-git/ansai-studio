# Development Guide

## Status

🚧 **ANSAI Studio is in early development**

This guide will be updated as we build the MVP.

## Prerequisites

- **Node.js:** 18+ (for VS Code extension)
- **Python:** 3.11+ (for runtime server)
- **VS Code:** Latest version
- **Git:** For version control

## Quick Start (Coming Soon)

```bash
# Clone the repository
git clone https://github.com/thebyrdman-git/ansai-studio.git
cd ansai-studio

# Install dependencies
npm install           # Extension dependencies
pip install -r requirements.txt  # Runtime dependencies

# Run in development mode
npm run dev          # Extension in watch mode
python runtime/server.py  # Runtime server
```

## Project Structure

```
ansai-studio/
├── extension/          # VS Code Extension (TypeScript)
│   ├── src/           # Source code
│   ├── package.json   # Extension manifest
│   └── tsconfig.json  # TypeScript config
│
├── runtime/           # Python Backend (FastAPI)
│   ├── server.py      # HTTP server
│   ├── auth.py        # Authentication
│   ├── executor.py    # Ansible runner
│   └── requirements.txt
│
├── ai/                # AI Integration
│   ├── ollama.py      # Ollama client
│   └── download.py    # Model manager
│
├── docs/              # Documentation
└── tests/             # Test suite
```

## Development Milestones

### ✅ Milestone 0: Foundation (Current)
- [x] Repository created
- [x] Project structure defined
- [x] Documentation framework
- [ ] Development environment setup

### 🚧 Milestone 1: Hello World (Week 1-2)
- [ ] VS Code extension skeleton
- [ ] Basic Python HTTP server
- [ ] Token-based authentication
- [ ] Extension ↔ Runtime communication

### 📋 Milestone 2: Playbook Runner (Week 3-6)
- [ ] Import Ansible project
- [ ] Display playbook list
- [ ] Execute playbooks
- [ ] Stream output to terminal

### 📋 Milestone 3: Infrastructure Graph (Week 7-12)
- [ ] Parse inventory
- [ ] Generate graph
- [ ] Interactive visualization

### 📋 Milestone 4: AI Brain (Week 13-16)
- [ ] Ollama integration
- [ ] Model download manager
- [ ] AI autocomplete
- [ ] Error analysis

## Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for contribution guidelines.

### Areas We Need Help

1. **TypeScript/VS Code Extensions**
   - Extension development
   - Webview UI
   - Command palette integration

2. **Python/FastAPI**
   - Backend API development
   - Ansible integration
   - State management

3. **UI/UX Design**
   - Infrastructure graph design
   - User flow optimization
   - Icon design

4. **Documentation**
   - Tutorials
   - API documentation
   - Examples

5. **Testing**
   - Unit tests
   - Integration tests
   - E2E tests

## Development Workflow

### Extension Development

```bash
cd extension

# Install dependencies
npm install

# Compile in watch mode
npm run watch

# Launch Extension Development Host (F5 in VS Code)
# - Opens new VS Code window with extension loaded
# - Hot reload on file changes
```

### Runtime Development

```bash
cd runtime

# Create virtual environment
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows

# Install dependencies
pip install -r requirements.txt

# Run server with hot reload
uvicorn server:app --reload --port 11434
```

### Running Tests

```bash
# Extension tests
cd extension
npm test

# Runtime tests
cd runtime
pytest
```

## Architecture Overview

ANSAI Studio uses a three-layer architecture:

```
Extension (TypeScript) ↔ Runtime (Python) ↔ AI (Ollama/Cloud)
```

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed information.

## Debugging

### Extension Debugging

1. Open `extension/` in VS Code
2. Press F5 to launch Extension Development Host
3. Set breakpoints in TypeScript code
4. View Debug Console for logs

### Runtime Debugging

1. Add print statements or use Python debugger
2. View terminal output
3. Check logs in `~/.ansai-studio/logs/`

## Code Style

### TypeScript
- Use ESLint + Prettier
- Follow VS Code extension best practices
- Async/await for promises

### Python
- Follow PEP 8
- Use type hints
- Docstrings for public APIs

## Testing Strategy

### Unit Tests
- Test individual functions/classes
- Mock external dependencies
- Fast execution

### Integration Tests
- Test component interactions
- Real Ansible execution
- Database/file operations

### E2E Tests
- Test complete user workflows
- Extension + Runtime + AI
- Automated UI testing

## Release Process (Coming Soon)

1. Update version in `package.json` and `setup.py`
2. Update CHANGELOG.md
3. Create Git tag
4. Build extension (`.vsix`)
5. Build runtime package
6. Create GitHub release
7. Publish to VS Code Marketplace

## Resources

### VS Code Extension Development
- [Extension API](https://code.visualstudio.com/api)
- [Extension Samples](https://github.com/microsoft/vscode-extension-samples)

### FastAPI
- [FastAPI Documentation](https://fastapi.tiangolo.com/)

### Ansible Runner
- [Ansible Runner Docs](https://ansible-runner.readthedocs.io/)

### Ollama
- [Ollama Documentation](https://ollama.ai/docs)

---

**Questions?** Open a discussion or issue on GitHub!

