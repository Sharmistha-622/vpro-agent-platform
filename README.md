# VPRO Agentic AI – RAG Study Assistant

An AI study assistant that answers students' questions from uploaded course documents. It uses Retrieval-Augmented Generation (RAG) on **AWS Bedrock**, with a LangGraph agent workflow, an MCP server, and a React frontend.

> Built as part of the **VPro Skills Agentic AI program**, using the program's
> base codebase ([toolsknowledge/vpro-agent-platform](https://github.com/toolsknowledge/vpro-agent-platform)).
> My work covered the AWS Bedrock and IAM configuration, local environment setup,
> debugging, and documentation described below.

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React (Vite) |
| Backend | FastAPI (Python 3.12) |
| Database | PostgreSQL 16 + pgvector (Docker) |
| Embeddings | Amazon Titan Text Embeddings V2 (AWS Bedrock) |
| LLM | Amazon Nova 2 Lite (AWS Bedrock, cross-region inference profile) |
| Orchestration | LangGraph, LangChain |
| Tools | MCP server |
| Evaluation | RAGAS, custom evaluation gates, pytest |

## How it works

1. An admin uploads a PDF, DOCX or TXT file from the Admin dashboard.
2. The backend splits it into sections and creates embeddings with **Titan Text Embeddings V2**.
3. The embeddings are stored in **PostgreSQL with pgvector**.
4. When a student asks a question, the LangGraph workflow embeds the question, finds the most similar sections, and asks **Nova 2 Lite** to write a grounded answer with the source document.

## My setup work

- **AWS IAM, least privilege:** created a dedicated IAM user (`vpro-agentic-ai-demo`) with an inline policy that only allows `bedrock:InvokeModel` on Titan Text Embeddings V2 and the Nova 2 Lite inference profile and its destination models. Credentials live in a separate AWS CLI profile (`vpro-agentic-ai`) and are never committed.
- **Policy fix:** the inference-profile ARN must contain *your own* AWS account ID, not the example one from the course notes.
- **Environment:** created `backend/.env` (not committed; see the template below) and used **Python 3.12** for the virtual environment, matching the CI configuration.
- **Python path fix:** started the backend with `venv/bin/python -m uvicorn` so the virtual environment is always used, not a global Python install.
- **CORS / port fix:** the backend only allows `http://localhost:5173`. If another app uses that port, Vite silently moves to 5174 and the chat shows "Failed to fetch". Running Vite with `--strictPort` prevents this.
- **AWS account troubleshooting:** a new AWS account returned `ValidationException: Operation not allowed` for Bedrock models. `aws bedrock get-foundation-model-availability` showed `authorizationStatus: NOT_AUTHORIZED`. This was resolved through AWS Support, with no code changes needed.

## Run it locally

### Prerequisites

- Docker Desktop
- Python 3.12
- Node.js 20+
- AWS CLI v2, with a profile that has Bedrock access in `us-east-1`

### 1. Configure the backend environment

Create `backend/.env`:

```env
DATABASE_URL=postgresql+psycopg://vpro_user:vpro_password@localhost:5433/vpro_agent_db
AWS_PROFILE=vpro-agentic-ai
AWS_REGION=us-east-1
BEDROCK_EMBEDDING_MODEL_ID=amazon.titan-embed-text-v2:0
EMBEDDING_DIMENSIONS=1024
BEDROCK_CHAT_MODEL_ID=us.amazon.nova-2-lite-v1:0
RAG_TOP_K=3
RAG_MAX_COSINE_DISTANCE=0.65
```

The database credentials match `docker-compose.yml` and are for local development only.

### 2. Start the four services (one terminal each)

**Database**

```bash
docker compose up -d
```

**Backend** (http://127.0.0.1:8000)

```bash
cd backend
python3.12 -m venv venv
venv/bin/python -m pip install -r requirements.txt
export AWS_PROFILE=vpro-agentic-ai
venv/bin/python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```

**MCP server** (http://127.0.0.1:8001)

```bash
cd backend
export AWS_PROFILE=vpro-agentic-ai
venv/bin/python -m app.mcp_server
```

**Frontend** (http://localhost:5173)

```bash
cd frontend
npm install
npm run dev -- --port 5173 --strictPort
```

### 3. Check that it works

- `curl http://127.0.0.1:8000/api/db-health` should return `PostgreSQL connected successfully`.
- Open http://localhost:5173, sign in to the Admin dashboard, upload a document, and ask a question such as *"Describe what this course guide covers"*.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Failed to fetch` in the chat | Backend not running, or frontend not on port 5173 | Start the backend; run Vite with `--port 5173 --strictPort` |
| `ModuleNotFoundError: No module named 'pydantic'` | A global `uvicorn` is used instead of the venv one | Use `venv/bin/python -m uvicorn ...` |
| `ValidationException: Operation not allowed` | Bedrock not authorized on the AWS account | Check with `aws bedrock get-foundation-model-availability`; contact AWS Support |
| `LangGraph RAG workflow failed` | A Bedrock or database call failed inside the workflow | Check the backend terminal and the health endpoints |

## Security notes

- `.env`, `venv/` and `node_modules/` are excluded through `.gitignore`.
- No AWS access keys are stored in this repository. Credentials stay in the local AWS CLI profile.
- The Bedrock IAM user can only invoke the two models the app needs.
