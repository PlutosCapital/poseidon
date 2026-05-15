# Poseidon Fix Prompt - LLM Connection Issue

## Problem Statement
Poseidon (running on port 5002, `/home/boss/poseidon/backend`) fails to connect to Prometheus Ollama endpoint (`http://94.130.18.137:11434/v1`) when making LLM calls. The same connection works via curl but fails in Python's httpx.

## Error Trace
```
openai.APIConnectionError: Connection error.
httpcore.ConnectError: [Errno 111] Connection refused
```

## Evidence
- `curl http://94.130.18.137:11434/v1/chat/completions` works from Atlas
- Python httpx/openai from Poseidon's venv fails with Connection Refused
- Same issue affects Hermes agent on Atlas

## Environment Details
- **Prometheus IP:** 94.130.18.137 (Hetzner)
- **Ollama Port:** 11434
- **Poseidon Location:** /home/boss/poseidon/backend
- **Python venv:** /home/boss/poseidon/backend/.venv (Python 3.11)
- **System Python:** /home/boss/.local/bin/python3.11

## Root Cause Hypothesis
The Python venv on Atlas has network restrictions that block outbound HTTP connections, while curl uses system network stack which works.

## Tasks to Fix

### 1. Diagnose Network Issue
- [ ] Test Python network from Poseidon's venv: `python3 -c "import httpx; print(httpx.get('http://94.130.18.137:11434/api/tags'))"`
- [ ] Compare with system Python: `/usr/bin/python3 -c "import httpx; print(httpx.get('http://94.130.18.137:11434/api/tags'))"`
- [ ] Check for proxy environment variables: `env | grep -i proxy`
- [ ] Check iptables/firewall rules on Atlas
- [ ] Try using requests library instead of httpx

### 2. Implement Fix Options
**Option A:** Configure HTTP_PROXY/HTTPS_PROXY in .env
```
HTTP_PROXY=http://94.130.18.137:11434
HTTPS_PROXY=http://94.130.18.137:11434
```

**Option B:** Run LLM calls via subprocess (like we did for testing):
```python
import subprocess
import json

def call_ollama_via_subprocess(prompt):
    cmd = [
        'curl', '-s', 'http://94.130.18.137:11434/v1/chat/completions',
        '-H', 'Content-Type: application/json',
        '-d', json.dumps({
            "model": "qwen2.5-coder:7b",
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": 4096
        })
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    return json.loads(result.stdout)['choices'][0]['message']['content']
```

**Option C:** Install system CA certificates in venv:
```bash
source .venv/bin/activate
pip install --upgrade certifi
```

**Option D:** Use localhost tunnel (SSH port forwarding):
```bash
ssh -L 11434:localhost:11434 prometheus
# Then connect to localhost:11434 instead of 94.130.18.137:11434
```

### 3. Test Fix
- [ ] Run ontology generation task via Poseidon API
- [ ] Compare output with MiroFish (5001)
- [ ] Verify graph structure matches

## Expected Output
Once fixed, this command should succeed:
```bash
curl -s -X POST http://localhost:5002/api/graph/ontology/generate \
  -F "files=@/tmp/circle_debate.txt" \
  -F "project_name=TEST-Circle-Poseidon-FIXED" \
  -F "simulation_requirement=Will Circle become next money monopoly?"
```

## Reference
- MiroFish (5001) successfully completed this task
- Working .env config in /home/boss/poseidon/.env:
```
LLM_API_KEY=ollama
LLM_BASE_URL=http://94.130.18.137:11434/v1
LLM_MODEL_NAME=qwen2.5-coder:7b
```