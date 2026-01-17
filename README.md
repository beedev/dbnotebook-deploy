# DBNotebook Deployment

Docker deployment for DBNotebook - A multimodal RAG system with NotebookLM-style document organization.

## Quick Start

### Prerequisites

- Docker and Docker Compose
- **OpenAI API key** (required for embeddings)
- **Groq API key** (recommended for fast LLM inference)
- Google API key (optional, for infographics generation)

### Deployment Steps

1. **Clone this repository**
   ```bash
   git clone https://github.com/beedev/dbnotebook-deploy.git
   cd dbnotebook-deploy
   ```

2. **Configure environment**
   ```bash
   cp .env.example .env
   ```

   Edit `.env` and update:
   - **API Keys**: Set `OPENAI_API_KEY` (required for embeddings), `GROQ_API_KEY` (recommended for LLM)
   - **Optional**: `GOOGLE_API_KEY` (for infographics), `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`
   - **Database name**: Change `POSTGRES_DB` if needed (default: `dbnotebook_debug`)

3. **Pull the latest image**

   For **Apple Silicon** (M1/M2/M3):
   ```bash
   docker pull ghcr.io/beedev/dbnotebook:latest-arm64
   ```

   For **Intel/AMD** (Linux, Windows):
   ```bash
   docker pull ghcr.io/beedev/dbnotebook:latest-amd64
   ```

4. **Configure models** (optional)

   Edit `config/models.yaml` to customize available models:
   - **Note**: GPT-5 is not supported yet
   - **Google API key** is required for infographics generation

5. **Create data directories**
   ```bash
   mkdir -p data outputs uploads
   ```

6. **Start fresh**
   ```bash
   docker compose down -v   # Clean up any existing containers/volumes
   docker compose up -d
   ```

7. **Access the application**

   Open http://localhost:7860 in your browser.

### First Use

1. Go to **RAG** section
2. Create a new **Notebook**
3. Upload your content (documents, PDFs, etc.)
4. **Wait a few minutes** for document processing and embedding
5. Select a model (e.g., **GPT-4.1**) and start chatting

> **Note:** Database migrations run automatically on container startup.

## LLM Providers

**Groq (Recommended)**
- Fast inference with Llama models
- Set `GROQ_API_KEY` in `.env`
- Models: `meta-llama/llama-4-maverick-17b-128e-instruct`, `llama-3.3-70b-versatile`
- Rate limit: ~300K tokens/min (use staggered requests for high concurrency)

**OpenAI**
- Set `OPENAI_API_KEY` in `.env`
- Models: `gpt-4.1-mini`, `gpt-4.1` (1M context), `gpt-4o`, `gpt-4o-mini`

**Anthropic**
- Set `ANTHROPIC_API_KEY` in `.env`
- Models: `claude-sonnet-4-20250514`, `claude-3-5-haiku-latest`

**Google Gemini**
- Set `GEMINI_API_KEY` in `.env`
- Models: `gemini-2.0-flash`, `gemini-1.5-pro`

> **Note:** Embeddings require OpenAI API key regardless of LLM provider.

## Configuration Files

The `config/` directory contains YAML configuration files:

| File | Description |
|------|-------------|
| `ingestion.yaml` | Document chunking, embedding, and retrieval settings |
| `models.yaml` | LLM and embedding model configurations |
| `raptor.yaml` | RAPTOR hierarchical retrieval parameters |
| `sql_chat.yaml` | SQL Chat feature configuration |

These files are mounted read-only into the container. Modify them to customize behavior.

## Updating

To update to the latest version:

```bash
docker compose pull
docker compose up -d
```

Migrations run automatically on startup - no manual steps needed.

## Database Migrations

Migrations run automatically when the container starts. To check status manually:

```bash
docker compose exec dbnotebook alembic current
docker compose exec dbnotebook alembic history
```

## Troubleshooting

### Container logs

```bash
docker compose logs -f dbnotebook
```

### Reset database

```bash
docker compose down -v  # Warning: deletes all data
docker compose up -d
```

---

## PostgreSQL + pgvector Setup

### Quick Start (Docker - Recommended)

DBNotebook includes PostgreSQL in its `docker-compose.yml`:

```bash
docker compose up -d
```

This automatically:
- Starts PostgreSQL 15 with pgvector 0.7.0
- Creates the database `dbnotebook`
- Exposes port 5433 (to avoid conflicts with local PostgreSQL)

### Standalone PostgreSQL with pgvector

```bash
docker run -d \
  --name pgvector-db \
  -e POSTGRES_USER=dbnotebook \
  -e POSTGRES_PASSWORD=dbnotebook \
  -e POSTGRES_DB=dbnotebook \
  -p 5433:5432 \
  pgvector/pgvector:pg16

# Connect with psql
docker exec -it pgvector-db psql -U dbnotebook -d dbnotebook
```

### Enable pgvector Extension

```sql
CREATE EXTENSION IF NOT EXISTS vector;
SELECT * FROM pg_extension WHERE extname = 'vector';
```

### Native Installation

<details>
<summary><strong>macOS (Homebrew)</strong></summary>

```bash
brew install postgresql@15
brew services start postgresql@15
brew install pgvector
createdb dbnotebook
psql -d dbnotebook -c "CREATE EXTENSION vector;"
```
</details>

<details>
<summary><strong>Ubuntu/Debian</strong></summary>

```bash
sudo apt install -y postgresql-15 postgresql-server-dev-15 build-essential git
git clone --branch v0.7.0 https://github.com/pgvector/pgvector.git
cd pgvector && make && sudo make install

sudo -u postgres psql
# CREATE USER dbnotebook WITH PASSWORD 'dbnotebook';
# CREATE DATABASE dbnotebook OWNER dbnotebook;
# \c dbnotebook
# CREATE EXTENSION vector;
```
</details>

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | - | Full connection URL |
| `POSTGRES_HOST` | `localhost` | PostgreSQL host |
| `POSTGRES_PORT` | `5432` | PostgreSQL port |
| `POSTGRES_DB` | `dbnotebook` | Database name |
| `POSTGRES_USER` | `dbnotebook` | Database user |
| `POSTGRES_PASSWORD` | `dbnotebook` | Database password |
| `PGVECTOR_EMBED_DIM` | `1536` | Embedding dimension |

---

## API Guide

### Authentication

All API requests require an `X-API-Key` header:

```bash
curl http://localhost:7860/api/query/notebooks \
  -H "X-API-Key: YOUR_API_KEY"
```

Default admin API key: `dbn_00000000000000000000000000000001`

### List Notebooks

```bash
curl http://localhost:7860/api/query/notebooks \
  -H "X-API-Key: YOUR_API_KEY"
```

### Execute a Query

```bash
curl -X POST http://localhost:7860/api/query \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "notebook_id": "YOUR_NOTEBOOK_UUID",
    "query": "What is the main topic?"
  }'
```

### Conversation Memory

The API supports multi-turn conversations. Send a `session_id` (UUID) to enable memory:

```bash
SESSION_ID=$(uuidgen)

# First query
curl -X POST http://localhost:7860/api/query \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d "{
    \"notebook_id\": \"YOUR_NOTEBOOK_UUID\",
    \"query\": \"What is the policy?\",
    \"session_id\": \"$SESSION_ID\"
  }"

# Follow-up (has context)
curl -X POST http://localhost:7860/api/query \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d "{
    \"notebook_id\": \"YOUR_NOTEBOOK_UUID\",
    \"query\": \"Can you elaborate?\",
    \"session_id\": \"$SESSION_ID\"
  }"
```

### Query Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `notebook_id` | UUID | Yes | Notebook to query |
| `query` | string | Yes | Natural language query |
| `session_id` | UUID | No | For conversation memory |
| `model` | string | No | LLM model override |
| `max_sources` | int | No | Max sources (1-20, default: 6) |
| `top_k` | int | No | Retrieval chunks (default: 6) |
| `reranker_enabled` | bool | No | Enable reranking (default: true) |
| `skip_raptor` | bool | No | Skip RAPTOR summaries (default: true) |

### Supported Models

| Provider | Models | Notes |
|----------|--------|-------|
| Groq | `meta-llama/llama-4-maverick-17b-128e-instruct`, `llama-3.3-70b-versatile` | Fast inference, recommended |
| OpenAI | `gpt-4.1-mini`, `gpt-4.1`, `gpt-4o`, `gpt-4o-mini` | gpt-4.1 has 1M context |
| Anthropic | `claude-sonnet-4-20250514`, `claude-3-5-haiku-latest` | |
| Gemini | `gemini-2.0-flash`, `gemini-1.5-pro` | |

### Python Example Script

Use the included example script for quick API testing:

```bash
# List available notebooks
python scripts/query_api_example.py --list-notebooks

# List available models
python scripts/query_api_example.py --list-models

# Query with a specific model
python scripts/query_api_example.py --model gpt-4.1-mini --query "What is the policy?"

# Run full demo
python scripts/query_api_example.py --model gpt-4o

# Single query only
python scripts/query_api_example.py -m gpt-4.1-mini -q "What is the leave policy?" --no-demo

# Advanced options
python scripts/query_api_example.py --model gpt-4.1-mini --top-k 10 --no-reranker
```

**Environment Variables:**
```bash
export DBNOTEBOOK_API_URL="http://localhost:7860"
export DBNOTEBOOK_API_KEY="your-api-key"
export DBNOTEBOOK_NOTEBOOK_ID="your-notebook-uuid"
```

---

## User Management & RBAC

### Roles

| Role | Description | Permissions |
|------|-------------|-------------|
| **admin** | Full system access | Manage users, roles, notebooks, connections |
| **user** | Standard access | Create notebooks/connections, view/edit assigned |
| **viewer** | Read-only | View assigned notebooks only |

### Default Admin

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `admin123` |
| API Key | `dbn_00000000000000000000000000000001` |

> **Security**: Change the default password in production.

### Authentication Endpoints

```bash
# Login
curl -X POST http://localhost:7860/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "admin123"}'

# Get current user
curl http://localhost:7860/api/auth/me \
  -H "X-API-Key: YOUR_API_KEY"

# Logout
curl -X POST http://localhost:7860/api/auth/logout
```

### User Management (Admin Only)

```bash
# Create user
curl -X POST http://localhost:7860/api/admin/users \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ADMIN_API_KEY" \
  -d '{
    "username": "john",
    "email": "john@example.com",
    "password": "securepassword",
    "roles": ["user"]
  }'

# List users
curl http://localhost:7860/api/admin/users \
  -H "X-API-Key: ADMIN_API_KEY"
```

### Notebook Access Control

| Access Level | Capabilities |
|--------------|--------------|
| `owner` | Full control (edit, delete, share) |
| `editor` | Edit documents, run queries |
| `viewer` | View documents, run queries (read-only) |

```bash
# Grant access
curl -X POST http://localhost:7860/api/admin/notebooks/<notebook_id>/access \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ADMIN_API_KEY" \
  -d '{"user_id": "...", "access_level": "editor"}'
```

### API Keys

- Prefixed with `dbn_` for identification
- Unique per user
- Can be regenerated anytime

```bash
# Regenerate own API key
curl -X POST http://localhost:7860/api/auth/api-key \
  -H "X-API-Key: YOUR_CURRENT_KEY"
```

---

## Security Best Practices

1. **Change default password** immediately in production
2. **Use strong passwords** with minimum length and complexity
3. **Rotate API keys** periodically
4. **Principle of least privilege** - assign minimum necessary roles
5. **Use HTTPS** in production
6. **Audit access** regularly

---

## License

Apache-2.0
