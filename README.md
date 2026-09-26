---
title: AI War Room
emoji: 🛡️
colorFrom: red
colorTo: green
sdk: docker
app_port: 7860
tags: [openenv, reinforcement-learning, cybersecurity, drone-defense, simulation, multi-agent, defense-ai]
---

# 🛡️ AI War Room — Cyber & Drone Defense Simulator

> **An OpenEnv-compatible reinforcement learning environment where an AI agent defends critical infrastructure against simultaneous cyber attacks and drone incursions in real time.**

🔴 **Live Demo:** [https://harshitkhanna16-ai-war-room.hf.space](https://harshitkhanna16-ai-war-room.hf.space)

📖 **API Docs:** [https://harshitkhanna16-ai-war-room.hf.space/docs](https://harshitkhanna16-ai-war-room.hf.space/docs)

<sub>⚠️ Before starting the live demo, reset the simulator/environment to its initial state.</sub>

---

## 🌍 Real-World Utility

Modern Security Operations Centers (SOCs) and defense command posts face multi-vector threats simultaneously — a cyber breach escalating in the background while hostile drones close in on physical assets. Human operators must triage, prioritize, and act with limited resources under extreme time pressure.

This environment models exactly that challenge:

- 🖥️ **Cyber threats** that escalate through stages: `probing → breach → critical`
- 🚁 **Drone threats** that physically close in over time and cause damage if not intercepted
- ⚙️ **Resource constraints** — limited cyber teams and defense units force hard prioritization
- 👁️ **Partial observability** — not all threats are visible at once, mimicking real sensor limitations
- ⏱️ **Time pressure** — delayed action leads to cascading damage and mission failure

This directly maps to real SOC decision-making workflows and can be used to train, evaluate, and benchmark autonomous defense agents.

---

## 🎮 Environment Overview

| Property | Value |
|---|---|
| **Environment ID** | `war_room` |
| **Action Space** | Multi-discrete (`idle`, `block cyber`, `intercept drone`) |
| **Observation Space** | Structured JSON — threats, resources, damage, time |
| **Reward Range** | `0.0 – 1.0` (normalized) |
| **Episode Length** | Variable by task (terminates early if damage ≥ 10) |
| **Tasks** | 3 — Easy → Medium → Hard |
| **Framework** | OpenEnv spec compliant |

---

## 🗂️ Action Space

Actions are sent as a list — one action per available resource unit per step.

```json
{
  "actions": ["block cyber", "intercept drone"]
}
```

| Action | Description |
|---|---|
| `idle` | Do nothing this step |
| `block cyber` | Deploy one cyber team to neutralize an active cyber threat |
| `intercept drone` | Deploy one defense unit to shoot down an incoming drone |

Multiple actions can be issued per step, but each consumes one resource unit. If no resources remain, additional actions are ignored — forcing the agent to learn resource-aware prioritization.

---

## 👁️ Observation Space

Full typed observation returned by `reset()`, `step()`, and `state()`:

```json
{
  "time": 5,
  "damage": 2,
  "resources": {
    "defense_units": 2,
    "cyber_teams": 1
  },
  "visible_threats": [
    {
      "id": 1,
      "type": "cyber",
      "stage": "breach",
      "status": "active"
    },
    {
      "id": 2,
      "type": "drone",
      "distance": 15,
      "speed": 4,
      "status": "active"
    }
  ],
  "threats": [
    {
      "id": 0,
      "type": "drone",
      "status": "resolved"
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `time` | int | Current step number |
| `damage` | int | Accumulated damage (0–10, episode ends at 10) |
| `resources.defense_units` | int | Available drone interceptors |
| `resources.cyber_teams` | int | Available cyber responders |
| `visible_threats` | list | Currently observable active threats (partial observability) |
| `threats` | list | Full threat history including resolved |

---

## 🏆 Tasks

### ✅ Easy — `POST /reset?task=easy`
- 1 drone threat, ample resources (2 defense units, 1 cyber team)
- 6 max steps, drone starts at distance 20
- **Expected agent score: 0.80 – 1.00**

### ⚠️ Medium — `POST /reset?task=medium`
- 3 simultaneous threats (2 drones + 1 cyber), 1 hidden
- Limited resources (1 defense unit, 1 cyber team), 8 max steps
- **Expected agent score: 0.50 – 0.75**

### 🔴 Hard — `POST /reset?task=hard`
- 4 threats (2 drones + 2 cyber), 2 hidden
- Extreme resource constraints (1 defense unit, 1 cyber team), 10 max steps
- Cyber breach already in progress at start
- **Expected agent score: 0.20 – 0.50**

---

## 🎯 Reward Function

Rewards are **dense and shaped** — the agent receives signal at every step, not just at episode end.

| Event | Reward Delta |
|---|---|
| Threat successfully neutralized | `+0.15` per threat |
| Action taken while threats active | `+0.10` bonus |
| Damage accumulated | `-0.10 × damage` |
| Critical cyber threat active | `-0.30` per threat |
| Drone within distance 5 | `-0.20` per threat |
| Cyber threat at breach stage | `-0.15` per threat |
| Idle while threats are active | `-0.10` |
| Time pressure | `-0.01 × step` |

Final score from `/score` is normalized to **[0.0, 1.0]** via the deterministic task grader.

---

## 🔌 API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Serves the live War Room UI |
| `POST` | `/reset?task=easy` | Reset env, returns initial observation |
| `POST` | `/step` | Take a step, returns `state + reward + done` |
| `GET` | `/state` | Get current state without stepping |
| `GET` | `/score` | Get final normalized score (0.0–1.0) |

### Quick Start — Python

```python
import requests

BASE = "https://harshitkhanna16-ai-war-room.hf.space"

# 1. Reset
resp = requests.post(f"{BASE}/reset?task=easy").json()
state = resp["state"]

# 2. Step loop
done = False
total_reward = 0
while not done:
    result = requests.post(f"{BASE}/step", json={
        "actions": ["block cyber", "intercept drone"]
    }).json()
    state = result["state"]
    total_reward += result["reward"]
    done = result["done"]

# 3. Get graded score
score = requests.get(f"{BASE}/score").json()
print(f"Final score: {score['score']:.3f}")
```

---

## 🤖 Baseline Inference Script

```bash
python inference.py
```

The baseline agent uses the **OpenAI client** (via `API_BASE_URL` + `MODEL_NAME` env vars) with a rule-based fallback if the LLM call fails.

### Required Environment Variables

```bash
export API_BASE_URL="https://api.openai.com/v1"
export MODEL_NAME="gpt-4o-mini"
export HF_TOKEN="your_token_here"
```

### Expected Output Format

```
[START] task=easy env=war_room model=gpt-4o-mini
[STEP] step=1 action=intercept_drone reward=0.24 done=false error=null
[STEP] step=2 action=block_cyber reward=0.30 done=false error=null
...
[END] success=true steps=6 score=0.85 rewards=[0.24,0.30,...]
```

### Baseline Scores

| Task | Max Steps | Expected Score |
|---|---|---|
| Easy | 6 | 0.80 – 1.00 |
| Medium | 8 | 0.50 – 0.75 |
| Hard | 10 | 0.20 – 0.50 |

---

## 🚀 Setup & Installation

### Prerequisites
- Python 3.10+
- Docker (for containerized run)

### Local Setup

```bash
# Clone
git clone https://huggingface.co/spaces/HarshitKhanna16/AI-War-Room
cd AI-War-Room

# Install
pip install -r requirements.txt

# Run
uvicorn backend.api.main:app --host 0.0.0.0 --port 7860
```

Open: `http://localhost:7860`

### Docker Setup

```bash
docker build -t ai-war-room .
docker run -p 7860:7860 ai-war-room
```

---

## 📁 Project Structure

```
AI-War-Room/
├── backend/
│   ├── api/
│   │   └── main.py          # FastAPI server + static frontend serving
│   ├── env/
│   │   ├── env.py           # Core OpenEnv environment logic
│   │   └── models.py        # Pydantic typed Action/Observation models
│   ├── grader/
│   │   ├── grader.py        # Deterministic task graders (easy/medium/hard)
│   │   └── reward.py        # Dense step-level reward function
│   ├── tasks/
│   │   ├── task_easy.py     # Easy task config
│   │   ├── task_medium.py   # Medium task config
│   │   └── task_hard.py     # Hard task config
│   ├── agent/
│   │   └── risk.py          # Risk scoring for threat prioritization
│   └── utils/
│       └── logger.py        # Optional step logging
├── frontend/
│   ├── index.html           # Live War Room command UI
│   ├── script.js            # Real-time backend-driven frontend logic
│   └── style.css            # Military-themed dark UI styling
├── Dockerfile               # Container definition
├── inference.py             # Baseline inference script (OpenAI client)
├── openenv.yaml             # OpenEnv metadata and task registry
├── requirements.txt         # Python dependencies
└── README.md                # This file
```

---

## ✅ OpenEnv Compliance Checklist

- ✅ `reset()` returns clean typed initial observation
- ✅ `step(action)` returns `(state, reward, done, info)`
- ✅ `state()` returns current observation without side effects
- ✅ Typed Pydantic models for `Action` and `Observation`
- ✅ `openenv.yaml` present at repo root with correct YAML syntax
- ✅ `inference.py` present at repo root using OpenAI client
- ✅ `Dockerfile` present — builds and runs cleanly on port 7860
- ✅ 3 tasks with deterministic graders (easy → medium → hard)
- ✅ All scores in `[0.0, 1.0]` range
- ✅ Dense reward signal throughout episode
- ✅ Deployed and running on Hugging Face Spaces (Docker)
- ✅ Episode terminates cleanly with `done=true`

---

## 🏗️ Built With

- **FastAPI** — High-performance async backend API
- **Pydantic** — Typed observation and action models
- **Uvicorn** — ASGI server
- **OpenAI SDK** — LLM agent inference
- **Docker** — Containerized deployment
- **Hugging Face Spaces** — Cloud hosting
- **Chart.js** — Live real-time metrics charts
- **Python 3.10** — Core environment logic

---

## 👤 Authors

**Manas Khanna** & **Harshit Khanna**
