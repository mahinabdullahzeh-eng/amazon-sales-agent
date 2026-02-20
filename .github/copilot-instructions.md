# GitHub Copilot Instructions

## Project Overview

This is an **Amazon Sales Intelligence Agent** — an AI-powered platform for Amazon product analysis and SEO optimization.

## Repository Structure

```
amazon-sales-agent/
├── backend/          # FastAPI Python backend
│   ├── app/
│   │   ├── api/      # REST API endpoints (upload, research keywords, background jobs)
│   │   ├── core/     # Configuration and settings
│   │   ├── db/       # Database layer (Upstash Redis)
│   │   ├── local_agents/  # OpenAI Agents SDK pipelines
│   │   ├── schemas/  # Pydantic models
│   │   └── services/ # Business logic (scraping, keyword analysis, SEO)
│   └── tests/        # pytest test suite
└── frontend/         # Next.js TypeScript frontend
    ├── app/          # Next.js App Router pages
    ├── components/   # React UI components
    └── lib/          # Utility functions
```

## Tech Stack

- **Backend**: FastAPI, Python 3.12, OpenAI Agents SDK, Playwright (Amazon scraping), Upstash Redis
- **Frontend**: Next.js, TypeScript, Tailwind CSS, shadcn/ui
- **CI/CD**: GitHub Actions, Render (deployment)

## Key Conventions

- Backend uses `uv` for dependency management (`pyproject.toml`)
- API versioning under `/api/v1/`
- Background jobs stored in Redis for async processing
- Keyword relevancy filtered dynamically (2/3/5 threshold based on dataset size)
- AI agents process keywords in batches of 100–200 to avoid timeouts
