# POSEIDON - Build From Scratch Instruction

## Project Overview
**Name:** Poseidon  
**Purpose:** Zero-cost self-hosted AI swarm simulation engine (MiroFish alternative)  
**License:** AGPL-3.0  
**Repo:** https://github.com/PlutosCapital/poseidon

---

## The Problem (Why We Rebuild)
MiroFish depends on **Zep Cloud** (paid GraphRAG service) for:
- Knowledge graph storage
- Entity/relationship extraction
- Agent memory management

**Goal:** Replace Zep with free local alternatives:
- NetworkX + SQLite (instead of Zep graph)
- ChromaDB (instead of Zep embeddings)  
- Ollama local LLM (instead of paid APIs)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Frontend (Vue 3 + D3.js)              │
│   Wizard UI → Graph Visualization → Chat Interface       │
└─────────────────────────┬───────────────────────────────┘
                          │ HTTP REST
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   Backend (Flask 3.x)                   │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐  │
│  │ Ontology     │  │ Graph Builder │  │ Simulation   │  │
│  │ Generator    │  │ (NetworkX)    │  │ Manager      │  │
│  └──────┬───────┘  └───────┬───────┘  └──────┬───────┘  │
│         │                  │                 │          │
│         ▼                  ▼                 ▼          │
│  ┌──────────────────────────────────────────────────┐   │
│  │           LLM Router (Ollama localhost)           │   │
│  │    → qwen2.5-coder:7b on Prometheus (94.130.18.137)│   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│            Persistence Layer (Zero-Cost)                 │
│  ┌──────────┐  ┌────────────┐  ┌─────────────────────┐  │
│  │ SQLite   │  │ NetworkX   │  │ Filesystem (JSON)    │  │
│  │(projects,│  │ graphs     │  │ (logs, uploads)     │  │
│  │ tasks)   │  │ serialized │  │                     │  │
│  └──────────┘  └────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Current Zep Dependencies (Files to Replace)

| File | Zep Usage | Replace With |
|------|-----------|--------------|
| `services/graph_builder.py` | Zep SDK for graph storage | NetworkX + SQLite |
| `services/zep_entity_reader.py` | Read entities from Zep | Local SQLite query |
| `services/zep_graph_memory_updater.py` | Update graph memory | NetworkX + JSON |
| `services/zep_tools.py` | Zep tools for agents | Custom local tools |
| `services/oasis_profile_generator.py` | Zep for persona gen | Local LLM only |
| `utils/zep_paging.py` | Paginate Zep queries | Local pagination |
| `api/simulation.py` | Zep entity reader | Local entity reader |
| `api/report.py` | Zep tools service | Custom tools |

---

## Environment & Configuration

### LLM Configuration (Ollama on Prometheus)
```
LLM_API_KEY=ollama
LLM_BASE_URL=http://94.130.18.137:11434/v1
LLM_MODEL_NAME=qwen2.5-coder:7b
```

### Infrastructure
- **Prometheus (Ollama):** 94.130.18.137:11434 (Hetzner, GTX 1080)
- **Atlas (Backend):** localhost, running Flask on port 5002
- **Python:** 3.11 (venv at /home/boss/poseidon/backend/.venv)

### Python Dependencies (current working stack)
```
flask>=3.0.0
flask-cors>=6.0.0
openai>=1.0.0
pydantic>=2.0.0
python-dotenv>=1.0.0
PyMuPDF>=1.24.0
charset-normalizer>=3.0.0
chardet>=5.0.0
networkx>=3.0
# NEW - for replacement:
chromadb>=0.5.0
sqlite3 (stdlib)
```

---

## Implementation Phases

### Phase 1: Core Scaffold (Day 1)
1. Create new backend structure:
   ```
   backend/app/
   ├── __init__.py
   ├── config.py
   ├── api/
   │   ├── __init__.py
   │   ├── graph.py      # NEW: NetworkX-based
   │   ├── simulation.py
   │   └── report.py
   ├── services/
   │   ├── __init__.py
   │   ├── graph_builder.py      # REPLACE: NetworkX
   │   ├── local_entity_reader.py # NEW
   │   ├── local_memory.py       # NEW
   │   ├── ontology_generator.py # KEEP: LLM-based
   │   ├── simulation_manager.py
   │   └── simulation_runner.py
   ├── models/
   │   ├── project.py
   │   └── task.py
   ├── utils/
   │   ├── llm_client.py
   │   ├── file_parser.py
   │   └── logger.py
   ```

2. Create local graph storage (`services/local_graph_store.py`):
```python
import sqlite3
import json
import networkx as nx
from typing import Dict, List, Any

class LocalGraphStore:
    """Replace Zep Cloud with local NetworkX + SQLite"""
    
    def __init__(self, db_path: str = "data/poseidon.db"):
        self.conn = sqlite3.connect(db_path)
        self._init_tables()
    
    def _init_tables(self):
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS projects (
                project_id TEXT PRIMARY KEY,
                name TEXT,
                status TEXT,
                ontology_json TEXT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS graphs (
                graph_id TEXT PRIMARY KEY,
                project_id TEXT,
                nx_graph_json TEXT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        self.conn.commit()
    
    def create_graph(self, project_id: str, name: str) -> str:
        graph_id = f"poseidon_{project_id[:8]}"
        G = nx.DiGraph()
        self.conn.execute(
            "INSERT INTO graphs (graph_id, project_id, nx_graph_json) VALUES (?, ?, ?)",
            (graph_id, project_id, json.dumps(nx.node_link_data(G)))
        )
        self.conn.commit()
        return graph_id
    
    def add_node(self, graph_id: str, node_type: str, **attrs):
        data = self._load_graph(graph_id)
        node_id = attrs.get('id', f"{node_type}_{len(data.nodes())}")
        data.add_node(node_id, type=node_type, **attrs)
        self._save_graph(graph_id, data)
    
    def add_edge(self, graph_id: str, source: str, target: str, edge_type: str, **attrs):
        data = self._load_graph(graph_id)
        data.add_edge(source, target, type=edge_type, **attrs)
        self._save_graph(graph_id, data)
    
    def get_graph_data(self, graph_id: str) -> Dict:
        """For frontend D3.js visualization"""
        G = self._load_graph(graph_id)
        nodes = [{"id": n, **G.nodes[n]} for n in G.nodes()]
        edges = [{"source": u, "target": v, **G[u][v]} for u, v in G.edges()]
        return {"nodes": nodes, "edges": edges}
    
    def _load_graph(self, graph_id: str) -> nx.DiGraph:
        row = self.conn.execute(
            "SELECT nx_graph_json FROM graphs WHERE graph_id = ?", (graph_id,)
        ).fetchone()
        if row and row[0]:
            return json_graph.node_link_graph(json.loads(row[0]))
        return nx.DiGraph()
    
    def _save_graph(self, graph_id: str, G: nx.DiGraph):
        self.conn.execute(
            "UPDATE graphs SET nx_graph_json = ? WHERE graph_id = ?",
            (json.dumps(json_graph.node_link_data(G)), graph_id)
        )
        self.conn.commit()
```

3. Create local entity reader (`services/local_entity_reader.py`):
```python
from typing import List, Dict, Any
from .local_graph_store import LocalGraphStore

class LocalEntityReader:
    """Replace Zep entity reading with local queries"""
    
    def __init__(self, graph_store: LocalGraphStore):
        self.graph_store = graph_store
    
    def get_entities(self, graph_id: str, entity_type: str = None) -> List[Dict]:
        """Get all entities, optionally filtered by type"""
        data = self.graph_store.get_graph_data(graph_id)
        entities = data["nodes"]
        if entity_type:
            entities = [e for e in entities if e.get("type") == entity_type]
        return entities
    
    def get_entity_by_name(self, graph_id: str, name: str) -> Dict:
        """Find entity by name attribute"""
        entities = self.get_entities(graph_id)
        for e in entities:
            if e.get("name") == name or e.get("full_name") == name:
                return e
        return None
    
    def get_related_entities(self, graph_id: str, entity_id: str, relation: str = None) -> List[Dict]:
        """Get entities connected to given entity"""
        data = self.graph_store.get_graph_data(graph_id)
        G = json_graph.node_link_graph(data)
        
        related = []
        for neighbor in G.neighbors(entity_id):
            edge_data = G[entity_id][neighbor]
            if relation is None or edge_data.get("type") == relation:
                related.append({"entity": G.nodes[neighbor], "relation": edge_data})
        return related
```

4. Create local memory manager (`services/local_memory.py`):
```python
import json
from datetime import datetime
from typing import List, Dict, Any
from pathlib import Path

class LocalMemoryManager:
    """Replace Zep memory with local JSON file storage"""
    
    def __init__(self, memory_dir: str = "data/memory"):
        self.memory_dir = Path(memory_dir)
        self.memory_dir.mkdir(exist_ok=True)
    
    def save_memory(self, agent_id: str, memories: List[Dict]):
        """Save agent memory to JSON"""
        path = self.memory_dir / f"{agent_id}.json"
        path.write_text(json.dumps(memories, indent=2))
    
    def load_memory(self, agent_id: str) -> List[Dict]:
        """Load agent memory from JSON"""
        path = self.memory_dir / f"{agent_id}.json"
        if path.exists():
            return json.loads(path.read_text())
        return []
    
    def add_memory(self, agent_id: str, memory: Dict):
        """Add single memory entry"""
        memories = self.load_memory(agent_id)
        memory["timestamp"] = datetime.utcnow().isoformat()
        memories.append(memory)
        self.save_memory(agent_id, memories)
    
    def search_memories(self, agent_id: str, query: str) -> List[Dict]:
        """Simple text search in memories"""
        memories = self.load_memory(agent_id)
        results = []
        for m in memories:
            text = json.dumps(m).lower()
            if query.lower() in text:
                results.append(m)
        return results
```

### Phase 2: API Updates (Day 2)
1. Update `app/__init__.py` to use new local services
2. Update all imports from `zep_*` to `local_*`
3. Update Flask blueprints to use NetworkX data format

### Phase 3: Integration (Day 3)
1. Connect ontology generator to local graph store
2. Connect simulation to local entity reader
3. Test end-to-end flow

### Phase 4: Testing & Polish (Day 4)
1. Run same benchmark task as MiroFish
2. Compare outputs
3. Fix gaps

---

## API Contracts to Maintain

Keep these endpoints working (just change internal implementation):

### Graph Blueprint (`/api/graph`)
- `POST /ontology/generate` → Generate ontology from docs
- `POST /build` → Build knowledge graph
- `GET /project/<id>` → Get project details
- `GET /data/<graph_id>` → Get graph for visualization
- `GET /task/<id>` → Get task status
- `DELETE /project/<id>` → Delete project

### Simulation Blueprint (`/api/simulation`)
- `POST /setup` → Setup simulation
- `POST /run` → Run simulation
- `GET /status/<id>` → Get simulation status
- `GET /agents/<id>` → Get agent profiles
- `GET /logs/<id>` → Get action logs

### Report Blueprint (`/api/report`)
- `POST /generate` → Generate report
- `POST /chat` → Chat with agent
- `GET /<id>` → Get report

---

## Key Files Reference

### Config
- Location: `backend/app/config.py`
- Current: Uses `.env` from project root

### LLM Client  
- Location: `backend/app/utils/llm_client.py`
- Uses OpenAI SDK with custom base_url for Ollama
- Works with Prometheus at 94.130.18.137:11434

### File Parser
- Location: `backend/app/utils/file_parser.py`
- Handles PDF, MD, TXT extraction

### Text Processor
- Location: `backend/app/services/text_processor.py`
- Chunks text for embedding

---

## Testing Benchmark

Use this task to verify implementation:

**Input:**
```text
Will Circle Internet Group be the next money monopoly besides Tether in regulated markets?

Bull thesis:
- Circle has regulatory approvals and partnerships with major banks
- USDC is fully reserved and transparent
- EU MiCA regulation favors compliant stablecoins

Bear thesis:
- Tether has massive network effects
- Circle lacks liquidity depth
- Competition from PYUSD, EURC
```

**Expected output:**
- 10 entity types (StablecoinIssuer, Regulator, Bank, etc.)
- 8 edge types (REGULATES, PARTNERS_WITH, COMPETES_WITH, etc.)
- Analysis summary in Chinese/English

---

## Notes

1. **Network Issue:** Python httpx in venv may have connection issues to Prometheus. Use subprocess curl wrapper if needed:
```python
def call_llm(prompt):
    result = subprocess.run([
        'curl', '-s', 'http://94.130.18.137:11434/v1/chat/completions',
        '-H', 'Content-Type: application/json',
        '-d', json.dumps({"model": "qwen2.5-coder:7b", "messages": [{"role": "user", "content": prompt}]})
    ], capture_output=True)
    return json.loads(result.stdout)['choices'][0]['message']['content']
```

2. **Keep OASIS/CAMEL-AI:** These are open-source and work with any OpenAI-compatible endpoint. Keep them, just point to local Ollama.

3. **Frontend:** Vue 3 + D3.js stays the same. Just needs updated API responses.

4. **Database:** Use SQLite for persistence. NetworkX graphs serialized as JSON blobs.

---

## Deliverables Checklist

- [ ] `services/local_graph_store.py` - NetworkX + SQLite replacement
- [ ] `services/local_entity_reader.py` - Entity reading from local DB
- [ ] `services/local_memory.py` - Agent memory with JSON files
- [ ] Updated imports in all API files
- [ ] Working `/api/graph/ontology/generate`
- [ ] Working `/api/graph/build`
- [ ] Working `/api/simulation/run`
- [ ] Benchmark test passes