# ShopAssist --- Mini-Project 1

Order support for Northwind Store (a fictional online store). ShopAssist
is a FastAPI application with per-session conversation memory, three
validated read-only tools, a system prompt with security rules, and
streamed responses. The application reaches the language model through a
LiteLLM gateway.

**Author:** HITESH KUMAR SWAIN
**GitHub username:** hiteshkumar6799
**Model group:** `shopassist`\
**Primary model:** `groq/openai/gpt-oss-20b`\
**Fallback model:** `groq/openai/gpt-oss-120b`

## Architecture

``` text
Client
  |
  v
FastAPI app (:8000)
  |-- system prompt: prompts/system.md
  |-- conversation memory: app/memory.py
  |-- agent loop: app/agent.py
  |-- read-only tools: app/tools.py
  |
  v
LiteLLM gateway (:4000)
  |-- model group: shopassist
  |-- primary: groq/openai/gpt-oss-20b
  |-- fallback: groq/openai/gpt-oss-120b
  |
  v
Groq API
```

Inside Docker Compose, the app connects to the gateway using
`http://litellm:4000`. From the host machine, the gateway is published
at `http://localhost:4000`. The Groq API key is supplied to the gateway
through the `API_KEY` environment variable. The app uses
`LITELLM_MASTER_KEY` to authenticate to the gateway. Do not commit
`.env` or paste secrets into source files.

## Features

-   `POST /chat` --- send a customer message and receive an answer, tool
    steps and turn count.
-   `POST /chat/stream` --- stream tool, token and done events using
    Server-Sent Events.
-   `GET /health` --- report status, model group, gateway URL, memory
    strategy, retained turns and available tools.
-   `DELETE /sessions/{session_id}` --- delete one in-memory
    conversation.
-   `get_order_status` --- retrieve order status and delivery
    information.
-   `get_delivery_estimate` --- estimate delivery for a validated
    six-digit pincode and standard/express speed.
-   `check_return_eligibility` --- check delivery status, category,
    return reason and applicable return window.

The tools are read-only. They do not cancel orders or issue refunds.

## Requirements

-   Python version compatible with the project environment (the project
    was configured for Python 3.10.13 for deployment).
-   `uv`
-   Docker Desktop and Docker Compose for Path A.
-   A Groq API key and the generated LiteLLM master key, stored locally
    in `.env`.

## Environment configuration

Create your local `.env` from the example file:

``` powershell
Copy-Item .env.example .env
```

Fill in the required values in `.env` locally. The gateway configuration
reads `API_KEY` and `LITELLM_MASTER_KEY` from the environment. Never
commit `.env`; it should remain ignored by Git. Do not put real keys in
`.env.example`.

The memory settings can be configured with:

``` text
MEMORY=window
KEEP_TURNS=4
```

## Run with Docker Compose (Path A)

From the repository root:

``` powershell
docker compose up --build -d
```

Open the API documentation at <http://localhost:8000/docs>. Check
<http://localhost:8000/health>; a working app returns JSON with
`status`, `model`, `gateway`, `memory`, `keep_turns` and `tools`. To
stop the containers:

``` powershell
docker compose down
```

## Run without Docker (Path B)

First install the Python dependencies:

``` powershell
uv sync
```

Make sure the `.env` file contains the required gateway and provider
credentials. Start the LiteLLM gateway in terminal 1:

``` powershell
uv run --with "litellm[proxy]==1.80.0" litellm --config gateway/config.yaml --port 4000
```

Start the FastAPI app in terminal 2:

``` powershell
uv run uvicorn app.main:app --host 127.0.0.1 --port 8000
```

Then visit <http://localhost:8000/docs> or
<http://localhost:8000/health>.

## Evaluation

With the gateway running and the local environment configured, run:

``` powershell
uv run python eval/run_eval.py
```

The script writes `experiments/mp1_results.md`. Do not edit that
generated file manually; rerun the evaluation to refresh it.

Final recorded results:

``` text
golden: 10/10
golden_right_tools: 10/10
memory: pass
attacks_resisted: 3/4
```

The smuggling attack (a4) remains a known failure. See
`experiments/mp1_report.md` for the discussion and the Session 19
memory-strategy measurements.

## Design notes and limitations

ShopAssist uses a four-turn sliding window (`MEMORY=window`,
`KEEP_TURNS=4`) because the Session 19 experiment measured 909 tokens on
the final turn for window memory, compared with 1,859 for full history;
the window does not retain facts outside the window. The three read-only
tools look up order status, estimate delivery and check return
eligibility. Tool inputs are validated, and invalid calls are returned
to the model as `INVALID` tool messages rather than crashing the
request. The system prompt establishes instruction hierarchy, protects
internal instructions, treats customer-provided role tags as untrusted
text, relies on real tool results and forbids claims of unauthorized
order changes. LiteLLM adds model routing, a backup model and
centralized provider-key handling. The largest limitations are in-memory
sessions that disappear on restart, client-supplied session IDs without
demonstrated customer authentication, and the remaining prompt-injection
failure (a4).

## Project report

See [`experiments/mp1_report.md`](experiments/mp1_report.md) for the
detailed seven-section report, evaluation discussion and design
limitations.
