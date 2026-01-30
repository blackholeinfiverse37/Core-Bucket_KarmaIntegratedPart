# BHIV Core - AI Decision Engine

**Part of**: BHIV Core ↔ Bucket ↔ Karma Integration System  
**Port**: 8002  
**Status**: ✅ PRODUCTION READY

An advanced multi-modal AI pipeline with UCB-based reinforcement learning, knowledge-base retrieval (multi-folder vector search + NAS + FAISS + file retriever), production-ready FastAPI layer, web interface, and enhanced CLI.

## 🔗 Integration Status

✅ **Bucket Integration**: Fire-and-forget event writes (2s timeout, async)  
✅ **Karma Integration**: Direct behavioral logging (2s timeout, async)  
✅ **Graceful Degradation**: Works independently if Bucket/Karma offline  
✅ **Zero Regression**: Original functionality 100% preserved

> **Note:** If you see a '0 vector stores' message during startup, this is normal when no FAISS indices are present. The system will automatically fall back to other retrieval methods in the multi-folder vector search pipeline.

## Key Features

### Core Capabilities
- **Multi-modal processing**: text, PDF, image, audio with automatic type detection
- **RL Agent Selection**: UCB algorithm with exploration decay (0.995 per task)
- **Knowledge Base**: Multi-folder vector search across 4 Qdrant folders with priority weighting
- **Integration Clients**: Fire-and-forget writes to Bucket + direct logging to Karma
- **Comprehensive Logging**: MongoDB + file logs + memory handler + RL replay buffer

### Advanced Features
- **UCB Algorithm**: Upper Confidence Bound for agent selection (better than epsilon-greedy)
- **Exploration Decay**: Automatic reduction from 20% to 5% minimum
- **Task Complexity Weighting**: Higher exploration for complex tasks (audio 2.5x, image 2.0x)
- **Agent Confidence Scoring**: Based on historical performance + experience
- **Timeout Protection**: All external calls have 2-second timeout
- **Async Operations**: Non-blocking throughout (Motor, aiohttp)

### Integration Features
- **Fire-and-Forget**: Never waits for Bucket/Karma responses
- **Optional Context**: Can read agent context from Bucket (non-blocking)
- **Dual-Path Logging**: Direct to Karma + via Bucket forwarding
- **Zero Latency Impact**: User response time unchanged (2-5s)

## What's New in Integration

- **Bucket Client**: Fire-and-forget event writes with 2s timeout
- **Karma Client**: Direct behavioral logging for Q-learning
- **RL Context**: Enhanced logging for all RL decisions
- **Agent Memory**: Persistent memory across requests
- **Multi-Folder Vector Search**: Search across 4 Qdrant folders simultaneously
- **UCB Agent Selection**: Smarter than epsilon-greedy exploration
- **Graceful Degradation**: Works even if Bucket/Karma offline

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Interface │    │   CLI Runner    │    │  Simple API     │
│   (Port 8003)   │    │  (Enhanced)     │    │  (Port 8001)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └──────────────┬────────┴──────────────┬────────┘
                        │                       │
                   ┌───────────────┐       ┌───────────────┐
                   │  MCP Bridge   │       │  Knowledge KB │
                   │  (Port 8002)  │       │  (Multi-Folder)│
                   └───────────────┘       └───────────────┘
                              │
                 ┌────────────┴────────────┐
                 │  Agent Registry + RL    │
                 │  (UCB-based selection)  │
                 └────────────┬─────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼────┐         ┌─────▼─────┐       ┌─────▼─────┐
    │ Bucket  │         │   Karma   │       │  MongoDB  │
    │ Client  │         │  Client   │       │  Logger   │
    │(async)  │         │  (async)  │       │           │
    └─────────┘         └───────────┘       └───────────┘
```

### Integration Flow

1. **Request** → MCP Bridge receives task
2. **Optional Read** → Read agent context from Bucket (2s timeout)
3. **RL Selection** → UCB algorithm selects best agent
4. **Execution** → Agent processes task (python module or HTTP API)
5. **Logging** → MongoDB + Memory + RL replay buffer
6. **Fire-and-Forget** → Write to Bucket (async, <100ms)
7. **Direct Log** → Write to Karma (async, <500ms)
8. **Response** → Return to user (2-5s total)

## Quick Start

### Prerequisites
- Python 3.11+
- MongoDB 5.0+ (required for logging/analytics)
- Optional: Qdrant for vector search (multi-folder support)
- Optional: Redis for caching
- Bucket service running on port 8001 (optional, for integration)
- Karma service running on port 8000 (optional, for integration)

### Install
```bash
git clone <repository-url>
cd BHIV-Third-Installment
python -m venv .venv && .venv\Scripts\activate  # Windows PowerShell
pip install -r requirements.txt
# Optional NLP model
python -m spacy download en_core_web_sm
```

### Environment
Create a `.env` file:
```env
# MongoDB (required)
MONGO_URI=mongodb://localhost:27017

# Reinforcement Learning (required)
USE_RL=true
RL_EXPLORATION_RATE=0.2  # Initial exploration rate (decays to 0.05)

# LLM API Keys (optional)
GROQ_API_KEY=your_key_if_used
GEMINI_API_KEY=your_key_if_used

# Single Qdrant instance (used as fallback)
QDRANT_URL=http://localhost:6333
QDRANT_COLLECTION=vedas_knowledge_base
QDRANT_VECTOR_SIZE=384

# Multi-Folder Vector Configuration (primary retrieval method)
QDRANT_URLS=http://localhost:6333  # Comma-separated URLs if using multiple servers
QDRANT_INSTANCE_NAMES=qdrant_data,qdrant_fourth_data,qdrant_legacy_data,qdrant_new_data

# Integration URLs (optional - defaults shown)
BUCKET_URL=http://localhost:8001
KARMA_URL=http://localhost:8000
```

> **Important**: The multi-folder vector configuration is essential for enabling comprehensive search across all your Qdrant data folders. Make sure both `QDRANT_URLS` and `QDRANT_INSTANCE_NAMES` are properly configured.

### Run Services (recommended ports)
```powershell
# Terminal 1: Simple API (agents, KB endpoints)
python simple_api.py --port 8001

# Terminal 2: MCP Bridge (task router, RL, registry)
python mcp_bridge.py  # serves on port 8002

# Terminal 3: Web Interface (auth: admin/secret, user/secret)
python integration/web_interface.py  # serves on port 8003
```

### Endpoints
- Web UI: `http://localhost:8003`
- MCP Bridge health: `http://localhost:8002/health`
- Simple API docs: `http://localhost:8001/docs`

## Usage

### CLI
```bash
# Text
python cli_runner.py summarize "Explain artificial intelligence" edumentor_agent --input-type text

# Single file (auto-detect type)
python cli_runner.py summarize "Analyze this file" edumentor_agent --file sample_documents/ayurveda_basics.txt

# Batch directory → save CSV
python cli_runner.py summarize "Process folder" edumentor_agent --batch ./sample_documents --output results.csv --output-format csv

# RL options
python cli_runner.py summarize "Learning test" edumentor_agent --use-rl --rl-stats --exploration-rate 0.3
```

### MCP Bridge API (port 8002)
```bash
# JSON task
curl -X POST http://localhost:8002/handle_task \
  -H "Content-Type: application/json" \
  -d '{"agent":"edumentor_agent","input":"Explain machine learning","input_type":"text"}'

# Multi-task
curl -X POST http://localhost:8002/handle_multi_task \
  -H "Content-Type: application/json" \
  -d '{"files":[{"path":"test.pdf","type":"pdf","data":"Analyze"}],"agent":"edumentor_agent","task_type":"summarize"}'
```

### Simple API (port 8001)
```bash
# Vedas
curl -X POST http://localhost:8001/ask-vedas -H "Content-Type: application/json" -d '{"query":"what is dharma"}'

# Educational
curl -X POST http://localhost:8001/edumentor -H "Content-Type: application/json" -d '{"query":"explain reinforcement learning"}'

# Wellness
curl -X POST http://localhost:8001/wellness -H "Content-Type: application/json" -d '{"query":"how to reduce stress"}'

# Knowledge Base (uses NAS → FAISS → file retriever fallback)
curl -X POST http://localhost:8001/query-kb -H "Content-Type: application/json" -d '{"query":"agent architecture"}'

# Health
curl http://localhost:8001/health
```

### Web Interface (port 8003)
- Login with Basic Auth (`admin/secret` or `user/secret`)
- Upload files at `/` → processed via MCP Bridge
- Dashboard at `/dashboard` → recent tasks, NLO stats
- Download NLOs: `/download_nlo/{task_id}?format=json`

### Demo Pipeline
```bash
python blackhole_demo.py  # edit defaults within to point to your input
```

## Configuration

### Core Configuration Files
- **Agent Registry**: `config/agent_configs.json` - Agent definitions and routing
- **RL Config**: `config/settings.py` (`RL_CONFIG`) - UCB parameters, exploration rate
- **Timeouts**: `config/settings.py` (`TIMEOUT_CONFIG`) - Per-input-type timeouts
- **Integration**: `integration/bucket_client.py`, `integration/karma_client.py`

### Key Configuration Options

**RL Configuration** (`config/settings.py`):
```python
RL_CONFIG = {
    "use_rl": True,  # Enable RL-based agent selection
    "exploration_rate": 0.2,  # Initial exploration (decays to 0.05)
    "exploration_decay": 0.995,  # Decay per task
    "min_exploration": 0.05  # Minimum exploration rate
}
```

**Agent Configuration** (`config/agent_configs.json`):
```json
{
  "knowledge_agent": {
    "connection_type": "python_module",
    "module_path": "agents.KnowledgeAgent",
    "class_name": "KnowledgeAgent",
    "tags": ["semantic_search", "vedabase"],
    "weight": 0.9
  }
}
```

**Timeout Configuration** (`config/settings.py`):
```python
TIMEOUT_CONFIG = {
    "default_timeout": 120,
    "image_processing_timeout": 180,
    "audio_processing_timeout": 240,
    "pdf_processing_timeout": 150,
    "bucket_timeout": 2.0,  # Fire-and-forget
    "karma_timeout": 2.0    # Fire-and-forget
}
```

## Multi-Folder Vector Search
The system now supports searching across multiple Qdrant folders simultaneously:

```
📁 Folder Structure:
  📂 qdrant_data
  📂 qdrant_fourth_data
  📂 qdrant_legacy_data
  📂 qdrant_new_data
```

### How It Works
1. System scans ALL folders for Qdrant collections
2. When you search, it queries EVERY collection in EVERY folder
3. Results are combined and ranked by relevance + folder priority
4. You get the BEST matches from your ENTIRE knowledge base

### Folder Priority Weights
- qdrant_new_data: 1.0 (highest priority)
- qdrant_fourth_data: 0.9
- qdrant_data: 0.8
- qdrant_legacy_data: 0.7

### Fallback Strategy
1. Multi-folder vector search
2. NAS+Qdrant retriever
3. Individual Qdrant retriever
4. FAISS vector stores
5. File-based retriever

### Understanding Startup Messages
When you see:
```
Initialized with 0 vector stores + multi-folder manager
```
This is normal and indicates:
- The system has successfully initialized the multi-folder vector manager
- No FAISS vector stores were loaded (they're only used as fallback)
- The system will use the multi-folder vector search as the primary retrieval method

The multi-folder vector manager will automatically discover and search across all available Qdrant collections in the configured folders.

## 📝 Notes

- Start Simple API on port 8001 to match `agent_configs.json` and `MODEL_CONFIG` endpoints
- For audio/image/PDF processing, ensure system deps like `ffmpeg`/`libsndfile` are available
- Core works independently even if Bucket/Karma are offline (graceful degradation)
- All integration writes are fire-and-forget (2s timeout, async)
- RL agent selection uses UCB algorithm (better than epsilon-greedy)
- Multi-folder vector search queries 4 Qdrant folders simultaneously
- Agent memory persists across requests for better context
- MongoDB logging is async and non-blocking

## 🚀 Production Deployment

See `../DEPLOYMENT_READY.md` for production deployment guide.

**Key Points**:
- Set `USE_RL=true` for production
- Configure MongoDB URI for persistence
- Set up Qdrant multi-folder for comprehensive search
- Ensure Bucket and Karma services are running
- Monitor health endpoints regularly
- Check integration stats periodically

## Testing

### Unit Tests
```bash
pytest -q
# Or run focused suites
pytest tests/test_web_interface_integration.py -q
```

### Integration Tests
```bash
# Test Core → Bucket → Karma integration
cd ..
python test_full_integration.py

# Expected: 5/6 tests passing (83%)
```

### Manual Testing
```bash
# Test RL agent selection
curl -X POST "http://localhost:8002/handle_task" \
  -H "Content-Type: application/json" \
  -d '{"agent": "edumentor_agent", "input": "Test RL", "input_type": "text"}'

# Test knowledge base query
curl -X POST "http://localhost:8002/query-kb" \
  -H "Content-Type: application/json" \
  -d '{"query": "What is dharma?"}'

# Check health
curl http://localhost:8002/health
```

## Troubleshooting

### Health Checks
```bash
# Core health
curl http://localhost:8002/health

# Check MongoDB connection
curl http://localhost:8002/health | jq '.services.mongodb'

# Check agent registry
curl http://localhost:8002/health | jq '.services.agent_registry'
```

### Common Issues

**Issue**: Core can't connect to Bucket
- ✅ **Solution**: Core continues normally, check Bucket is running on port 8001
- Check: `curl http://localhost:8001/health`

**Issue**: RL agent selection not working
- ✅ **Solution**: Check `USE_RL=true` in `.env`
- Check: `curl http://localhost:8002/config | jq '.rl_enabled'`

**Issue**: Multi-folder vector search not finding results
- ✅ **Solution**: Verify `.env` has both `QDRANT_URLS` and `QDRANT_INSTANCE_NAMES`
- Test: `python test_multi_folder_search.py`
- Check logs for: "[SUCCESS] Multi-folder vector manager initialized successfully"

**Issue**: "0 vector stores" message on startup
- ✅ **Solution**: This is NORMAL - refers to FAISS indices, not Qdrant collections
- Multi-folder vector search is still active

**Issue**: Timeout errors on large files
- ✅ **Solution**: Increase timeouts in `config/settings.py` (`TIMEOUT_CONFIG`)

**Issue**: MongoDB connection errors
- ✅ **Solution**: Check `MONGO_URI` in `.env`
- Test: `mongosh "$MONGO_URI"`

### Port Conflicts (Windows)
```bash
netstat -ano | findstr :8002  # Core
netstat -ano | findstr :8001  # Simple API
netstat -ano | findstr :8003  # Web UI
```

### Integration Debugging
```bash
# Check if Bucket is receiving events
curl http://localhost:8001/core/events

# Check integration stats
curl http://localhost:8001/core/stats

# Check Karma is receiving events
curl http://localhost:8000/api/v1/analytics/karma_trends
```

### Fallback Chain
If multi-folder vector search fails, system falls back to:
1. NAS+Qdrant retriever
2. Individual Qdrant retriever
3. FAISS vector stores (if present under `vector_stores/`)
4. File-based retriever (over `local_setup/storage/documents/`)

## 📚 Additional Documentation

- **COMPREHENSIVE_ARCHITECTURE_ANALYSIS.md** - Complete system architecture
- **Integration Guide**: See main README.md in parent directory
- **API Contract**: `../core_bucket_contract.md`
- **RL Documentation**: `README_RL.md`

## 🔗 Related Services

- **Bucket**: `../BHIV_Central_Depository-main/` (Port 8001)
- **Karma**: `../karma_chain_v2-main/` (Port 8000)
- **Integration Tests**: `../test_full_integration.py`

## 📊 Performance

- **Response Time**: 2-5 seconds (unchanged from standalone)
- **Bucket Write**: <100ms (async, fire-and-forget)
- **Karma Write**: <500ms (async, fire-and-forget)
- **User Impact**: 0ms (all integration is async)
- **RL Overhead**: <10ms (UCB calculation)
- **MongoDB Logging**: <50ms (async)

## 🎯 Integration Success Indicators

✅ Core starts without errors  
✅ Health check shows MongoDB connected  
✅ Agent registry loaded (check `/config`)  
✅ RL enabled (check `/health`)  
✅ Tasks process normally (2-5s response)  
✅ Events written to Bucket (check Bucket `/core/events`)  
✅ Behavioral data logged to Karma (check Karma `/api/v1/analytics/karma_trends`)  
✅ Multi-folder vector search operational (check logs)  
✅ Fire-and-forget pattern working (no timeout errors)  
✅ Zero regression (original functionality preserved)python test_multi_folder_search.py`
- To see a demo of the multi-folder system: `python demo_multi_folder.py`

## What's New in Third Installment
- Multi-folder vector search system for comprehensive knowledge retrieval across all Qdrant instances
- Unified Simple API with knowledge-base endpoints and model providers
- Reinforcement Learning context/logging integration in MCP Bridge
- Enhanced CLI with batch processing and multiple output formats
- Web UI with authentication, uploads, and NLO download

—

For advanced usage and deployment, see:
- `docs/complete_usage_guide.md`
- `docs/deployment.md`
- `example/quick_setup_guide.md`
