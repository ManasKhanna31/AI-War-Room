# AI War Room — Project Report

## Overview
AI War Room is a defensive simulation environment that combines cyber security and drone defense into a single OpenEnv-compatible reinforcement learning task. The project is designed to train and evaluate agents that must allocate limited resources, manage evolving threats, and minimize cumulative damage under time pressure.

## Key Components

### Backend
- `backend/env/env.py`
  - Implements `WarRoomEnv`, the environment API.
  - Supports `reset()`, `step(action)`, `state()`, and `get_score()`.
  - Uses deterministic seeding and task-specific episode length.
  - Injects new threats periodically and reveals hidden threats over time.

- `backend/env/simulator.py`
  - Advances threat state each step.
  - Drones move toward the target and cause damage when they arrive.
  - Cyber threats escalate from `probing` → `breach` → `critical`.
  - Actions consume available resources and can resolve active threats.

- `backend/env/models.py`
  - Defines `Observation`, `Action`, and `Reward` Pydantic models.
  - Provides safe validation and shape enforcement for API payloads.

- `backend/tasks/`
  - Defines three tasks: `easy`, `medium`, `hard`.
  - Each task includes initial threat state, resources, difficulty, and max steps.

- `backend/grader/reward.py`
  - Computes step-level reward with dense shaping.
  - Penalizes damage, active critical threats, breached cyber threats, and close drones.
  - Rewards resolved threats and non-idle actions.
  - Clips reward to `[0.0, 1.0]`.

- `backend/grader/grader.py`
  - Implements task-specific final scoring.
  - Uses deterministic grading logic for each difficulty.
  - Normalizes final score to `0.01–0.99`.

- `backend/api/main.py`
  - FastAPI application exposing `/reset`, `/step`, `/state`, `/score`, and frontend routes.
  - Uses a global environment instance.

### Server
- `server/app.py`
  - Launches the FastAPI app on `0.0.0.0:7860`.

### Frontend
- `frontend/index.html`
- `frontend/script.js`
- `frontend/style.css`
  - UI dashboard for environment visualization and logs.
  - Fetches data from the backend and displays threat metrics, charts, and logs.
  - Automatically adapts for Hugging Face hosting or local development.

### Inference / Agent
- `inference.py`
  - Runs a baseline agent on `easy`, `medium`, and `hard`.
  - Uses OpenAI chat completion with a fallback rule-based agent.
  - Prints structured step logs required by the hackathon spec.

## OpenEnv Specification
- `openenv.yaml`
  - Declares environment ID `war_room`.
  - Lists actions: `intercept drone`, `block cyber`, `idle`.
  - Lists observations: `visible_threats`, `threats`, `resources`, `time`, `damage`.
  - Defines reward as normalized and evaluation as deterministic grader-based.
  - States environment is deterministic with variable max steps by task.

## Tasks Summary

### Easy
- One drone threat.
- Ample resources: 2 defense units, 1 cyber team.
- Max steps: 6.
- Focus: intercept drone before damage.

### Medium
- Three threats: 2 drones + 1 cyber, one hidden.
- Limited resources: 1 defense unit, 1 cyber team.
- Max steps: 8.
- Focus: prioritize visible threats, uncover hidden threats, and minimize damage.

### Hard
- Four threats: 2 drones + 2 cyber, two hidden.
- Severe resource constraints: 1 defense unit, 1 cyber team.
- Max steps: 10.
- Focus: strategic prioritization and managing critical cyber escalation.

## Reward Function
The reward function in `backend/grader/reward.py` includes:
- Damage penalty: `-0.1` per damage (up to `-0.5`)
- Resolved threat reward: `+0.15` each (capped at `+0.5`)
- Active critical cyber penalty: `-0.3` each
- Drone close penalty: `-0.2` if distance ≤ 5
- Cyber breach penalty: `-0.15`
- Action bonus: `+0.1` for non-idle actions when threats exist
- Idle penalty: `-0.1`
- Time pressure penalty: `-0.01` per step (capped at `-0.2`)
- Final reward clipped to `[0.0, 1.0]`

## API Endpoints
- `POST /reset?task=<easy|medium|hard>` — initialize the environment
- `POST /step` — submit `{"actions": ["block cyber", "intercept drone"]}`
- `GET /state` — query current environment state
- `GET /score` — retrieve normalized final score

## How to Run
1. Install dependencies:
   - `pip install -r requirements.txt`
2. Start the server:
   - `python server/app.py`
3. Use the API from Python or HTTP clients.

## Notes
- The environment is deterministic by seed and task definition.
- The API stores a single global environment instance, so concurrent sessions are not isolated.
- `inference.py` uses environment variables:
  - `API_BASE_URL`
  - `MODEL_NAME`
  - `HF_TOKEN` or `OPENAI_API_KEY`

## Files of Interest
- `README.md` — high-level project description and usage.
- `openenv.yaml` — environment metadata for OpenEnv compatibility.
- `backend/env/env.py` — core environment loop.
- `backend/grader/reward.py` — reward shaping logic.
- `backend/grader/grader.py` — final score evaluation.
- `backend/api/main.py` — REST API endpoints.
- `inference.py` — baseline LLM + rule-based agent runner.
- `frontend/` — interactive dashboard assets.
