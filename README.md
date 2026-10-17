<div align="center">

# DevMate LLM API

A FastAPI backend for a tool-calling personal assistant with streaming replies, layered memory, and guardrails.

![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-Async%20API-009688?style=flat-square&logo=fastapi&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-Chat%20Completions-412991?style=flat-square&logo=openai&logoColor=white)
![Valkey](https://img.shields.io/badge/Valkey-Short--term%20memory-6983FF?style=flat-square&logoColor=white)
![NeMo Guardrails](https://img.shields.io/badge/NeMo-Guardrails-76B900?style=flat-square&logo=nvidia&logoColor=white)

</div>

## Overview

DevMate is a personal assistant exposed as an HTTP API. A client sends chat messages, and the model decides on its own when to check the date, search the web, or manage the user's to-do list. Short-term conversation history is kept in Valkey, durable facts about each user are extracted in the background and saved to JSON, and NeMo Guardrails screen messages on the way in and replies on the way out. Every reply reports its token usage and dollar cost.

## Key Features

- **Chat endpoint**: `POST /chat` returns the reply together with per-turn and per-session token counts and cost.
- **Streaming**: `GET /chat/stream` delivers the reply over Server-Sent Events as JSON events (`content`, `done`, `error`), so line breaks inside the model output do not break the stream.
- **Tool calling**: the model can call six tools: current date/time with a day offset, Tavily web search (up to 3 attempts, 5 results), and per-user to-do add, list, complete and delete. Token usage from both model calls is summed when a tool is used.
- **Layered memory**: the last 10 turns (20 messages) per user and session are kept in Valkey, and a background task asks the model to extract durable facts about the user, de-duplicates them, and stores them in a per-user JSON file.
- **Guardrails**: NeMo Guardrails self-check flows run on every input and output using the policy in `app/guardrails/prompts.yml`. User text is also wrapped in `<user_input>` tags, and the system prompt tells the model to ignore override attempts.
- **Cost tracking**: cost is computed from `INPUT_PRICE` and `OUTPUT_PRICE` (per 1M tokens) and accumulated per session.
- **Validated requests and consistent errors**: temperature, top_p and max_tokens are range-checked, blank messages are rejected, and failures come back as `{"error": ..., "message": ...}` with 422, 502 or 500.

## Tech Stack

| Area | Technology |
|---|---|
| Language | Python 3.12 |
| Web framework | FastAPI, Uvicorn |
| LLM | OpenAI Python SDK (Chat Completions, default model `gpt-4.1-mini`) |
| Safety | NeMo Guardrails (self check input and self check output) |
| Memory | Valkey (short-term history), per-user JSON files (long-term facts, to-do lists) |
| HTTP client | httpx (shared async client for Tavily search) |
| Validation and config | Pydantic, pydantic-settings |

## Project Structure

```text
app/
  main.py        # FastAPI app, router, validation error handler
  config/        # Settings loaded from environment / .env
  lifespan/      # Startup and shutdown: OpenAI, httpx, Valkey, guardrails
  routes/        # POST /chat and GET /chat/stream
  schemas/       # Request and response models
  services/      # LLM calls, memory, session totals, guardrail checks
  tools/         # Date/time, Tavily search and to-do tools
  prompts/       # System prompt and <user_input> wrapper
  guardrails/    # NeMo Guardrails config.yml and prompts.yml
  utils/         # Cost math, error shape, logging setup
data/            # Created at runtime: per-user memory and to-do JSON files
tests/
  manual_test_requests.py   # Live smoke test, one function per phase
```

## Getting Started

### Prerequisites

- Python 3.12 (NeMo Guardrails does not support Python 3.14, so a 3.12 virtual environment is recommended)
- A running Valkey or Redis instance (Docker is the simplest way to get one)
- An [OpenAI API key](https://platform.openai.com/)
- A [Tavily API key](https://tavily.com/)

### Installation

```bash
git clone https://github.com/shahramshafiq/devmate-llm-api.git
cd devmate-llm-api

# Windows (PowerShell)
py -3.12 -m venv venv
.\venv\Scripts\Activate.ps1

# macOS / Linux
# python3.12 -m venv venv
# source venv/bin/activate

pip install -r requirements.txt
cp .env.example .env
```

Fill in `.env`. The app reads these environment variables:

| Variable | Purpose |
|---|---|
| `OPENAI_API_KEY` | Required. OpenAI API key |
| `OPENAI_MODEL` | Chat model, defaults to `gpt-4.1-mini` |
| `INPUT_PRICE`, `OUTPUT_PRICE` | Required. Price per 1M input and output tokens, used for cost tracking |
| `TAVILY_API_KEY` | Required. Tavily API key for the web search tool |
| `TAVILY_TIMEOUT` | Tavily request timeout in seconds, defaults to `10.0` |
| `VALKEY_HOST`, `VALKEY_PORT` | Valkey location, default `localhost` and `6379` |

The guardrail checks use the model set in `app/guardrails/config.yml`, which is separate from `OPENAI_MODEL`.

### Run

Start Valkey, then start the API from the repository root (the guardrails config and `data/` folder are resolved relative to it):

```bash
docker run -p 6379:6379 valkey/valkey
python -m uvicorn app.main:app --reload
```

The API is served at `http://127.0.0.1:8000`, with interactive docs at `http://127.0.0.1:8000/docs`.

## Usage

Send a chat message (`temperature`, `top_p` and `max_tokens` are optional):

```bash
curl -X POST http://127.0.0.1:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"user_id": "demo", "session_id": "s1", "message": "Add a task to buy groceries"}'
```

The response contains `reply`, `turn` (`turn_input_tokens`, `turn_output_tokens`, `turn_cost`) and `session` (`session_input_tokens`, `session_output_tokens`, `session_cost`).

Stream a reply over Server-Sent Events (query parameters, not a JSON body):

```bash
curl -N "http://127.0.0.1:8000/chat/stream?user_id=demo&session_id=s1&message=Say%20hello"
```

Each event is a line of the form `data: {...}` with a `type` of `content`, `done` (includes `turn` and `session`) or `error`.

## Testing

There is no automated test suite yet. `tests/manual_test_requests.py` is a live smoke-test script with one function per phase: basic chat and validation, streaming, tools, memory, and guardrails (five jailbreak categories). It sends real requests to a running server, which in turn calls OpenAI and Tavily, so it costs real money to run.

```bash
pip install requests   # used by the script, not listed in requirements.txt
python tests/manual_test_requests.py            # all phases
python tests/manual_test_requests.py --phase 3  # one phase
```

## Limitations and Roadmap

- `/chat/stream` currently supports plain chat and cost tracking only, with no tools, memory or guardrails.
- Session cost totals are held in process memory and reset when the server restarts.
- To-do lists and long-term memory are plain per-user JSON files protected by an in-process lock, and the endpoints have no authentication, so this is suited to single-process local use.
- Mocked automated tests are the next gap to close.

## Author

**Shahram Shafiq** - [GitHub](https://github.com/shahramshafiq) | [LinkedIn](https://www.linkedin.com/in/shahramshafiq/)
