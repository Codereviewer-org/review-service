# Review Service

FastAPI service for storing code review results, generating repository-level analysis, and coordinating full GitHub repository reviews with the AI service.

The service persists review history and analysis data in PostgreSQL, exposes health and Prometheus-style metrics endpoints, and can clone public GitHub repositories for static scanning before sending a capped review payload to `ai-service`.

## Features

- Save, list, and delete review results by user.
- Analyze Python code for maintainability, complexity, memory, performance, health, and optimization recommendations.
- Start asynchronous full-repository review jobs for public GitHub HTTPS repositories.
- Track repository review job status and fetch completed results.
- Expose saved analysis bundles for dashboard use.
- Provide `/health`, `/ready`, `/live`, and `/metrics` endpoints for platform checks.

## Tech Stack

- Python 3.10
- FastAPI
- Uvicorn
- PostgreSQL through `psycopg2`
- Git, used at runtime for repository cloning

## Project Structure

```text
.
|-- analyzers/
|   `-- code_metrics.py          # AST-based code metrics and recommendations
|-- migrations/
|   `-- 001_repository_analysis.sql
|-- API_CONTRACTS.md             # Analysis endpoint response contract notes
|-- Dockerfile                   # Multi-stage non-root runtime image
|-- main.py                      # FastAPI app and persistence layer
|-- repository_review.py         # GitHub clone, scan, AI payload, and result flow
`-- requirements.txt
```

## Configuration

The service reads values from environment variables first. If a variable is missing, it also checks a matching secret file path such as `/mnt/secrets-store/DATABASE_URL`.

| Variable | Default | Description |
| --- | --- | --- |
| `DATABASE_URL` | `postgresql://coderaptor:coderaptor@postgres:5432/coderaptor` | PostgreSQL connection string. |
| `AI_SERVICE_URL` | `http://ai-service:8003` | Base URL for the AI service repository review endpoint. |
| `LOG_LEVEL` | `INFO` | Python logging level. |
| `REPOSITORY_REVIEW_WORKERS` | `2` | Number of background workers for repository review jobs. |
| `REPOSITORY_REVIEW_MAX_FILE_BYTES` | `200000` | Maximum supported file size during repository scans. |
| `REPOSITORY_REVIEW_MAX_FILES` | `250` | Maximum number of files included in a repository scan. |
| `REPOSITORY_REVIEW_MAX_AI_FILE_CHARS` | `12000` | Per-file character cap for AI payloads. |
| `REPOSITORY_REVIEW_MAX_AI_TOTAL_CHARS` | `90000` | Total character budget for AI payloads. |
| `REPOSITORY_REVIEW_MAX_UI_FILE_CHARS` | `30000` | Per-file character cap returned to the UI. |

Create a local environment file from the example when needed:

```powershell
Copy-Item .env.example .env
```

## Run Locally

Start PostgreSQL first, then install dependencies and run the API:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:DATABASE_URL = "postgresql://coderaptor:coderaptor@localhost:5432/coderaptor"
$env:AI_SERVICE_URL = "http://localhost:8003"
uvicorn main:app --host 0.0.0.0 --port 8004 --reload
```

Open the service docs at:

```text
http://localhost:8004/docs
```

## Run With Docker

Build and run the service image:

```powershell
docker build -t review-service .
docker run --rm -p 8004:8004 `
  -e DATABASE_URL="postgresql://coderaptor:coderaptor@host.docker.internal:5432/coderaptor" `
  -e AI_SERVICE_URL="http://host.docker.internal:8003" `
  review-service
```

The Docker image installs `git` and `ca-certificates`, exposes port `8004`, and runs Uvicorn as a non-root user.

## API Overview

### Platform Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/health` | Basic service health check. |
| `GET` | `/ready` | Checks PostgreSQL readiness. |
| `GET` | `/live` | Liveness check. |
| `GET` | `/metrics` | Prometheus-style request counters and latency total. |

### Saved Reviews

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/reviews/{username}` | Returns saved reviews for a user. |
| `POST` | `/reviews/{username}` | Saves or updates a review and persists analysis. |
| `DELETE` | `/reviews/{tab_id}` | Deletes a saved review by id. |

Example save request:

```powershell
Invoke-RestMethod -Method Post `
  -Uri "http://localhost:8004/reviews/demo-user" `
  -ContentType "application/json" `
  -Body '{
    "id": "example-review",
    "code": "def hello():\n    return \"world\"",
    "review_output": "Looks good.",
    "run_output": "world",
    "fixed_code": "def hello():\n    return \"world\"",
    "fixed_code_language": "python",
    "timestamp": "2026-06-28T00:00:00Z"
  }'
```

### Repository Review Jobs

Supported modes:

- `Security Review`
- `Performance Review`
- `Best Practices Review`
- `DevOps Review`
- `Full Repository Review`

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/review/repository` | Starts an async review job for a public GitHub repository. |
| `GET` | `/review/status/{job_id}` | Returns job status, progress, errors, and suggestions. |
| `GET` | `/review/result/{job_id}` | Returns completed repository review results. |

Example:

```powershell
$job = Invoke-RestMethod -Method Post `
  -Uri "http://localhost:8004/review/repository" `
  -ContentType "application/json" `
  -Body '{
    "repository_url": "https://github.com/example/project",
    "username": "demo-user",
    "mode": "Full Repository Review"
  }'

Invoke-RestMethod "http://localhost:8004/review/status/$($job.job_id)"
Invoke-RestMethod "http://localhost:8004/review/result/$($job.job_id)"
```

### Analysis Endpoints

These endpoints use the saved review id as `repository_id`.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/metrics/{repository_id}` | AST metrics, variable analysis, and function analysis. |
| `GET` | `/api/memory/{repository_id}` | Memory score, detected patterns, and recommendations. |
| `GET` | `/api/performance/{repository_id}` | Performance score, nested loop count, and recommendations. |
| `GET` | `/api/health/{repository_id}` | Weighted health score and component ratings. |
| `GET` | `/api/recommendations/{repository_id}` | Flattened optimization recommendations. |
| `GET` | `/api/analysis/{repository_id}` | Dashboard-friendly analysis bundle. |

See [API_CONTRACTS.md](API_CONTRACTS.md) for sample analysis responses.

## Database

`main.py` creates required tables on startup if they do not already exist:

- `reviews`
- `repository_metrics`
- `memory_analysis`
- `performance_analysis`
- `health_scores`
- `optimization_recommendations`
- `repository_review_jobs`

The `migrations/001_repository_analysis.sql` file contains the repository analysis schema for environments that apply migrations separately.

## Development Notes

- Repository review supports public HTTPS GitHub URLs like `https://github.com/user/project`.
- The service clones repositories with shallow Git clone options and scans supported source, config, and pipeline files.
- If AI service calls fail during a repository review, local scan results are still returned with `ai_error`.
- The readiness endpoint depends on a working PostgreSQL connection.
