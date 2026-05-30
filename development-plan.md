# Loan Origination System — Phased Development Plan

> Project: 219-loan-origination-system · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (Backend) | Python 3.12+ | Domain is data-heavy with significant AI/ML integration (document extraction, explainable decisioning, fraud detection). Python's ecosystem for LLMs (openai, anthropic), ML explainability (shap, lime), and financial computation (decimal, pandas) is unmatched. FastAPI provides async performance competitive with Node.js for API workloads. |
| API Framework | FastAPI 0.115+ | Native OpenAPI 3.1 schema generation (required by standards.md), async support for credit bureau and LLM calls, Pydantic v2 for request/response validation with JSON Schema generation, dependency injection for auth/RBAC. |
| Database | PostgreSQL 16 | JSONB support for product-specific variability (hybrid model), GIN indexes for JSONB queries, partitioning for audit logs, row-level security for multi-tenant isolation, TIMESTAMPTZ for regulatory timestamp compliance. Industry standard for financial applications. |
| ORM / Query Builder | SQLAlchemy 2.0 + Alembic | Mature async support, explicit SQL when needed for complex HMDA/compliance queries, Alembic for versioned migrations with rollback capability. |
| Task Queue | Celery 5+ with Redis broker | Async workloads: credit bureau pulls, document OCR/AI extraction, adverse action notice generation, notification delivery, HMDA batch reporting. Redis as broker for low-latency task dispatch; PostgreSQL as result backend for auditability. |
| Cache | Redis 7+ | Session cache, rate limiting on credit pulls, API response caching for product catalogue and workflow templates. |
| Frontend | Next.js 15 (React 19) + TypeScript | Server-side rendering for loan officer dashboards, React Server Components for data-heavy views, TypeScript for type safety across API contract. Shadcn/ui component library for consistent financial-grade UI. |
| Object Storage | S3-compatible (MinIO for self-hosted) | Document storage with server-side encryption, presigned URLs for secure upload/download, lifecycle policies for retention compliance. |
| Containerisation | Docker + Docker Compose | Self-hosted deployment target for community banks. Multi-stage builds for production images. Docker Compose for local development with all dependencies. |
| CI/CD | GitHub Actions | Automated testing, linting, type checking, Docker builds, migration validation. |
| Testing (Backend) | pytest + pytest-asyncio + factory_boy | pytest for unit/integration, factory_boy for test data factories, httpx for async API testing, testcontainers-python for PostgreSQL integration tests. |
| Testing (Frontend) | Vitest + Playwright | Vitest for component/unit tests, Playwright for E2E browser tests against the full stack. |
| Code Quality | Ruff (lint + format) + mypy (strict) | Ruff replaces black/isort/flake8 with 10-100x speed. mypy strict mode catches type errors at the boundary between API routes and business logic. |
| API Documentation | Auto-generated OpenAPI 3.1 | FastAPI generates the spec from Pydantic models; served via Swagger UI and ReDoc. Aligns with standards.md requirement for OAS 3.1 compliance. |
| Authentication | OAuth 2.0 + JWT | OAuth 2.0 for machine-to-machine API integrations (credit bureaus, core banking). JWT for staff/user sessions. OpenID Connect for SSO with institutional identity providers. Aligns with Fannie Mae, Freddie Mac, and Open Banking authentication requirements. |
| LLM Integration | Anthropic Claude API (primary) + OpenAI API (fallback) | Document extraction, adverse action notice generation, income analysis, borrower chatbot. Anthropic for primary due to longer context windows for multi-page documents. Provider-agnostic interface for flexibility. |
| Key Libraries | pydantic (validation), python-jose (JWT), passlib (password hashing), boto3 (S3), httpx (async HTTP), jinja2 (disclosure templates), reportlab (PDF generation), pytesseract + pdf2image (OCR), xmltodict (MISMO XML), openpyxl (Excel export) |
| Package Manager | uv (Python), pnpm (Node.js) | uv for fast, deterministic Python dependency resolution. pnpm for efficient Node.js monorepo management. |

### Project Structure

```
loan-origination-system/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── migrations/
│   └── versions/
├── src/
│   ├── __init__.py
│   ├── main.py                      # FastAPI app factory
│   ├── config.py                    # Pydantic Settings
│   ├── database.py                  # SQLAlchemy engine/session
│   ├── models/                      # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── party.py
│   │   ├── application.py
│   │   ├── collateral.py
│   │   ├── credit.py
│   │   ├── document.py
│   │   ├── underwriting.py
│   │   ├── compliance.py
│   │   ├── workflow.py
│   │   ├── organisation.py
│   │   └── audit.py
│   ├── schemas/                     # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── party.py
│   │   ├── application.py
│   │   ├── collateral.py
│   │   ├── credit.py
│   │   ├── document.py
│   │   ├── underwriting.py
│   │   ├── compliance.py
│   │   └── workflow.py
│   ├── api/                         # FastAPI routers
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── applications.py
│   │   │   ├── parties.py
│   │   │   ├── documents.py
│   │   │   ├── underwriting.py
│   │   │   ├── compliance.py
│   │   │   ├── workflow.py
│   │   │   ├── products.py
│   │   │   ├── auth.py
│   │   │   └── webhooks.py
│   │   └── deps.py                  # Dependency injection (auth, db session)
│   ├── services/                    # Business logic layer
│   │   ├── __init__.py
│   │   ├── application_service.py
│   │   ├── party_service.py
│   │   ├── credit_service.py
│   │   ├── document_service.py
│   │   ├── underwriting_service.py
│   │   ├── compliance_service.py
│   │   ├── workflow_service.py
│   │   ├── notification_service.py
│   │   └── audit_service.py
│   ├── integrations/                # External system connectors
│   │   ├── __init__.py
│   │   ├── credit_bureau.py
│   │   ├── document_ai.py
│   │   ├── esignature.py
│   │   ├── llm_provider.py
│   │   ├── storage.py
│   │   └── notification.py
│   ├── engine/                      # Decisioning and rules engine
│   │   ├── __init__.py
│   │   ├── decision_engine.py
│   │   ├── rules.py
│   │   ├── scoring.py
│   │   └── explainability.py
│   └── tasks/                       # Celery async tasks
│       ├── __init__.py
│       ├── credit_tasks.py
│       ├── document_tasks.py
│       ├── notification_tasks.py
│       └── compliance_tasks.py
├── frontend/
│   ├── package.json
│   ├── pnpm-lock.yaml
│   ├── next.config.ts
│   ├── tsconfig.json
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── (auth)/
│   │   │   ├── dashboard/
│   │   │   ├── applications/
│   │   │   ├── parties/
│   │   │   ├── documents/
│   │   │   └── compliance/
│   │   ├── components/
│   │   │   ├── ui/                  # Shadcn components
│   │   │   ├── application/
│   │   │   ├── party/
│   │   │   ├── document/
│   │   │   └── dashboard/
│   │   ├── lib/
│   │   │   ├── api-client.ts
│   │   │   ├── auth.ts
│   │   │   └── utils.ts
│   │   └── types/
│   │       └── api.ts               # Generated from OpenAPI spec
│   └── tests/
│       ├── e2e/
│       └── components/
├── tests/
│   ├── conftest.py
│   ├── factories/
│   │   ├── __init__.py
│   │   ├── party_factory.py
│   │   ├── application_factory.py
│   │   └── document_factory.py
│   ├── unit/
│   │   ├── test_decision_engine.py
│   │   ├── test_compliance.py
│   │   └── test_workflow.py
│   ├── integration/
│   │   ├── test_api_applications.py
│   │   ├── test_api_parties.py
│   │   └── test_credit_integration.py
│   └── fixtures/
│       ├── sample_application.json
│       ├── sample_credit_report.json
│       └── sample_documents/
├── templates/
│   ├── adverse_action_notice.html
│   ├── loan_estimate.html
│   └── closing_disclosure.html
└── docs/
    └── openapi.json                 # Auto-generated
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the project skeleton, development environment, database connection, configuration management, and CI pipeline. After this phase, developers can run the application locally, connect to PostgreSQL, run migrations, and execute the test suite. No business logic yet — this phase ensures every subsequent phase has a stable foundation to build on.

### Tasks

#### 1.1 — Project Initialisation and Configuration

**What**: Create the Python project with dependency management, configuration, and environment setup.

**Design**:

```python
# src/config.py
from pydantic_settings import BaseSettings
from pydantic import Field, PostgresDsn, RedisDsn

class Settings(BaseSettings):
    model_config = {"env_prefix": "LOS_", "env_file": ".env"}

    # Application
    app_name: str = "Loan Origination System"
    debug: bool = False
    api_version: str = "v1"
    host: str = "0.0.0.0"
    port: int = 8000

    # Database
    database_url: PostgresDsn = "postgresql+asyncpg://los:los@localhost:5432/los"
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: RedisDsn = "redis://localhost:6379/0"

    # Storage
    storage_backend: str = "s3"  # s3, local
    s3_bucket: str = "los-documents"
    s3_endpoint_url: str | None = None  # For MinIO
    s3_access_key: str = ""
    s3_secret_key: str = ""

    # Security
    secret_key: str = Field(min_length=32)
    jwt_algorithm: str = "HS256"
    jwt_expire_minutes: int = 480  # 8 hours
    bcrypt_rounds: int = 12

    # LLM
    anthropic_api_key: str = ""
    openai_api_key: str = ""
    llm_provider: str = "anthropic"  # anthropic, openai

    # External Services
    equifax_api_url: str = ""
    equifax_api_key: str = ""
    experian_api_url: str = ""
    experian_api_key: str = ""
    transunion_api_url: str = ""
    transunion_api_key: str = ""
    docusign_api_url: str = ""
    docusign_integration_key: str = ""
```

Environment file (`.env.example`):
```
LOS_SECRET_KEY=change-me-to-a-32-char-minimum-secret
LOS_DATABASE_URL=postgresql+asyncpg://los:los@localhost:5432/los
LOS_REDIS_URL=redis://localhost:6379/0
LOS_DEBUG=true
LOS_S3_ENDPOINT_URL=http://localhost:9000
LOS_S3_ACCESS_KEY=minioadmin
LOS_S3_SECRET_KEY=minioadmin
LOS_ANTHROPIC_API_KEY=
```

**Testing**:
- Unit: `Settings` loads correctly from environment variables with all defaults applied
- Unit: `Settings` raises `ValidationError` when `secret_key` is missing or too short
- Unit: `Settings` reads from `.env` file when present
- Unit: `database_url` validates as a proper PostgreSQL DSN

#### 1.2 — Database Connection and Session Management

**What**: Set up SQLAlchemy 2.0 async engine, session factory, and base model class.

**Design**:

```python
# src/database.py
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase
from datetime import datetime, timezone
from uuid import uuid4
import sqlalchemy as sa

class Base(DeclarativeBase):
    pass

class TimestampMixin:
    created_at: sa.orm.Mapped[datetime] = sa.orm.mapped_column(
        sa.DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        nullable=False,
    )
    updated_at: sa.orm.Mapped[datetime] = sa.orm.mapped_column(
        sa.DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
        nullable=False,
    )

class UUIDMixin:
    id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.Uuid, primary_key=True, default=uuid4
    )

# Engine and session factory created at app startup
engine = None
async_session_factory = None

async def init_db(database_url: str, pool_size: int = 20, max_overflow: int = 10):
    global engine, async_session_factory
    engine = create_async_engine(
        str(database_url),
        pool_size=pool_size,
        max_overflow=max_overflow,
        echo=False,
    )
    async_session_factory = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncSession:
    async with async_session_factory() as session:
        yield session
```

**Testing**:
- Integration: `init_db` connects to PostgreSQL test database successfully
- Integration: `get_session` yields a working async session that can execute `SELECT 1`
- Unit: `TimestampMixin` sets `created_at` and `updated_at` to UTC timestamps
- Unit: `UUIDMixin` generates unique UUIDs for new records

#### 1.3 — FastAPI Application Factory and Health Check

**What**: Create the FastAPI application with CORS, versioned routing, and a health check endpoint.

**Design**:

```python
# src/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
from src.config import Settings
from src.database import init_db

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = Settings()
    await init_db(str(settings.database_url), settings.database_pool_size)
    yield

def create_app() -> FastAPI:
    settings = Settings()
    app = FastAPI(
        title=settings.app_name,
        version=settings.api_version,
        lifespan=lifespan,
        docs_url="/api/docs",
        redoc_url="/api/redoc",
        openapi_url="/api/openapi.json",
    )
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["http://localhost:3000"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    # Register routers in later phases
    app.include_router(health_router, prefix="/api")
    return app

# Health check
from fastapi import APIRouter, Depends
from sqlalchemy import text
from src.database import get_session

health_router = APIRouter(tags=["health"])

@health_router.get("/health")
async def health_check(session=Depends(get_session)):
    await session.execute(text("SELECT 1"))
    return {"status": "healthy", "version": "0.1.0"}
```

**Testing**:
- Integration: `GET /api/health` returns `200` with `{"status": "healthy"}`
- Integration: `GET /api/docs` returns Swagger UI HTML
- Integration: `GET /api/openapi.json` returns valid OpenAPI 3.1 JSON
- Unit: `create_app()` returns a FastAPI instance with CORS middleware configured

#### 1.4 — Alembic Migration Setup

**What**: Configure Alembic for async PostgreSQL migrations.

**Design**:

`alembic.ini` configured with async driver. `migrations/env.py` imports all models from `src.models` so autogenerate detects schema changes. Migrations use `TIMESTAMPTZ` for all timestamp columns.

Alembic env.py must:
- Import `Base` from `src.database`
- Import all model modules to register them with the metadata
- Use `asyncpg` driver for async migration execution
- Support both `upgrade` and `downgrade` for every migration

**Testing**:
- Integration: `alembic upgrade head` runs without errors on a fresh database
- Integration: `alembic downgrade base` reverses all migrations cleanly
- Integration: `alembic check` reports no pending model changes after migration

#### 1.5 — Docker and Docker Compose

**What**: Create multi-stage Dockerfile and Docker Compose for local development.

**Design**:

```dockerfile
# Dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
RUN pip install uv

FROM base AS deps
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM base AS dev
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen
COPY . .
CMD ["uv", "run", "uvicorn", "src.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000", "--reload"]

FROM deps AS prod
COPY src/ src/
COPY migrations/ migrations/
COPY alembic.ini .
CMD ["uv", "run", "uvicorn", "src.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

```yaml
# docker-compose.yml
services:
  api:
    build:
      context: .
      target: dev
    ports: ["8000:8000"]
    env_file: .env
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
    volumes: ["./src:/app/src", "./tests:/app/tests"]

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: los
      POSTGRES_USER: los
      POSTGRES_PASSWORD: los
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U los"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes: ["miniodata:/data"]

volumes:
  pgdata:
  miniodata:
```

**Testing**:
- Integration: `docker compose up -d` starts all services without errors
- Integration: API container health check passes within 30 seconds
- Integration: `docker compose exec api uv run pytest tests/ -x` runs test suite inside container

#### 1.6 — CI Pipeline

**What**: GitHub Actions workflow for linting, type checking, and testing on every push/PR.

**Design**:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run ruff check src/ tests/
      - run: uv run ruff format --check src/ tests/
      - run: uv run mypy src/

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_DB: los_test, POSTGRES_USER: los, POSTGRES_PASSWORD: los }
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 5s --health-timeout 3s --health-retries 5
      redis:
        image: redis:7
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run pytest tests/ -v --tb=short
        env:
          LOS_DATABASE_URL: postgresql+asyncpg://los:los@localhost:5432/los_test
          LOS_SECRET_KEY: test-secret-key-minimum-32-characters
```

**Testing**:
- CI: Pipeline passes on clean checkout with all checks green
- CI: Lint failures block merge (enforced by branch protection)

---

## Phase 2: Core Data Model & CRUD API

### Purpose
Implement the database schema (based on the hybrid relational + JSONB model from data-model-suggestion-3) and expose CRUD endpoints for the core entities: parties, loan products, loan applications, and the application-party junction. After this phase, the system can store and retrieve loan applications with associated borrowers.

### Tasks

#### 2.1 — Organisation, Branch, and Staff Models

**What**: Create SQLAlchemy models, Pydantic schemas, and CRUD endpoints for organisation, branch, and staff entities.

**Design**:

```python
# src/models/organisation.py
import sqlalchemy as sa
from src.database import Base, UUIDMixin, TimestampMixin

class Organisation(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "organisation"
    name: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(255), nullable=False)
    lei: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(20))
    nmls_id: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(20))
    organisation_type: sa.orm.Mapped[str] = sa.orm.mapped_column(
        sa.String(30), nullable=False  # bank, credit_union, fintech, broker
    )
    org_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
    is_active: sa.orm.Mapped[bool] = sa.orm.mapped_column(default=True)

class Branch(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "branch"
    organisation_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("organisation.id"), nullable=False
    )
    branch_name: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(100), nullable=False)
    branch_code: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(20))
    nmls_id: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(20))
    address: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
    is_active: sa.orm.Mapped[bool] = sa.orm.mapped_column(default=True)

class Staff(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "staff"
    organisation_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("organisation.id"), nullable=False
    )
    branch_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(
        sa.ForeignKey("branch.id")
    )
    email: sa.orm.Mapped[str] = sa.orm.mapped_column(
        sa.String(255), unique=True, nullable=False
    )
    first_name: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(100), nullable=False)
    last_name: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(100), nullable=False)
    nmls_id: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(20))
    role: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), nullable=False)
    permissions: sa.orm.Mapped[list] = sa.orm.mapped_column(sa.JSON, default=list)
    password_hash: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(255), nullable=False)
    is_active: sa.orm.Mapped[bool] = sa.orm.mapped_column(default=True)
```

```python
# src/schemas/organisation.py
from pydantic import BaseModel, EmailStr, Field
from uuid import UUID
from datetime import datetime

class OrganisationCreate(BaseModel):
    name: str = Field(max_length=255)
    lei: str | None = Field(None, max_length=20)
    nmls_id: str | None = Field(None, max_length=20)
    organisation_type: str = Field(pattern="^(bank|credit_union|fintech|broker)$")

class OrganisationResponse(BaseModel):
    id: UUID
    name: str
    lei: str | None
    nmls_id: str | None
    organisation_type: str
    is_active: bool
    created_at: datetime

class StaffCreate(BaseModel):
    email: EmailStr
    first_name: str = Field(max_length=100)
    last_name: str = Field(max_length=100)
    role: str = Field(pattern="^(loan_officer|processor|underwriter|closer|compliance_officer|admin)$")
    nmls_id: str | None = None
    branch_id: UUID | None = None
    password: str = Field(min_length=12)

class StaffResponse(BaseModel):
    id: UUID
    email: str
    first_name: str
    last_name: str
    role: str
    nmls_id: str | None
    is_active: bool
    created_at: datetime
```

API endpoints:
- `POST /api/v1/organisations` — Create organisation
- `GET /api/v1/organisations` — List organisations (paginated)
- `GET /api/v1/organisations/{id}` — Get organisation by ID
- `PATCH /api/v1/organisations/{id}` — Update organisation
- `POST /api/v1/organisations/{org_id}/branches` — Create branch
- `GET /api/v1/organisations/{org_id}/branches` — List branches
- `POST /api/v1/organisations/{org_id}/staff` — Create staff member
- `GET /api/v1/organisations/{org_id}/staff` — List staff
- `GET /api/v1/staff/{id}` — Get staff by ID
- `PATCH /api/v1/staff/{id}` — Update staff

**Testing**:
- Unit: `OrganisationCreate` rejects invalid `organisation_type` values
- Unit: `StaffCreate` validates email format and password minimum length
- Integration: `POST /api/v1/organisations` creates organisation and returns 201 with UUID
- Integration: `GET /api/v1/organisations` returns paginated list with correct total count
- Integration: Create staff with non-existent `branch_id` returns 404
- Integration: Duplicate staff email returns 409 Conflict

#### 2.2 — Party Model and CRUD

**What**: Implement the party entity (individuals and organisations) with JSONB-based flexible data storage.

**Design**:

```python
# src/models/party.py
class Party(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "party"
    party_type: sa.orm.Mapped[str] = sa.orm.mapped_column(
        sa.String(20), nullable=False  # individual, organisation
    )
    display_name: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(255), nullable=False)
    email: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(255))
    phone_primary: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(20))
    ssn_last_four: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(4))
    status: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), default="active")
    party_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)

    __table_args__ = (
        sa.Index("idx_party_data", "party_data", postgresql_using="gin",
                 postgresql_ops={"party_data": "jsonb_path_ops"}),
        sa.CheckConstraint("party_type IN ('individual', 'organisation')"),
    )
```

```python
# src/schemas/party.py
class IndividualPartyData(BaseModel):
    """JSONB payload for individual party."""
    first_name: str
    middle_name: str | None = None
    last_name: str
    suffix: str | None = None
    date_of_birth: date | None = None
    citizenship: str | None = None
    marital_status: str | None = None
    dependents_count: int | None = None
    addresses: list[AddressData] = []
    employment: list[EmploymentData] = []
    income_sources: list[IncomeSourceData] = []
    assets: list[AssetData] = []
    liabilities: list[LiabilityData] = []

class AddressData(BaseModel):
    type: str  # current, mailing, previous
    street: str
    city: str
    state: str = Field(max_length=2)
    postal_code: str = Field(max_length=10)
    county_fips: str | None = None
    census_tract: str | None = None
    residence_type: str | None = None  # own, rent, living_rent_free
    years_at_address: float | None = None

class EmploymentData(BaseModel):
    employer_name: str
    position_title: str | None = None
    employment_type: str  # W2, self_employed, 1099, military, retired
    start_date: date
    end_date: date | None = None
    is_current: bool = True
    monthly_income: Decimal | None = None
    verified: bool = False
    verification_source: str | None = None
    verified_at: datetime | None = None

class PartyCreate(BaseModel):
    party_type: Literal["individual", "organisation"]
    email: EmailStr | None = None
    phone_primary: str | None = None
    party_data: IndividualPartyData | OrganisationPartyData

class PartyResponse(BaseModel):
    id: UUID
    party_type: str
    display_name: str
    email: str | None
    phone_primary: str | None
    ssn_last_four: str | None
    status: str
    party_data: dict
    created_at: datetime
    updated_at: datetime
```

API endpoints:
- `POST /api/v1/parties` — Create party
- `GET /api/v1/parties` — List/search parties (paginated, filterable by name, email, ssn_last_four)
- `GET /api/v1/parties/{id}` — Get party by ID
- `PATCH /api/v1/parties/{id}` — Update party (merge JSONB)
- `DELETE /api/v1/parties/{id}` — Soft-delete party (set status=inactive)

`display_name` is auto-computed from `party_data`: `"{first_name} {last_name}"` for individuals, `"{legal_name}"` for organisations.

**Testing**:
- Unit: `PartyCreate` validates discriminated union between individual and organisation data
- Unit: `display_name` computed correctly for individual ("Jane Smith") and organisation ("Acme LLC")
- Unit: `AddressData` rejects state codes longer than 2 characters
- Integration: Create individual party with full employment and address data, retrieve and verify JSONB structure
- Integration: Search parties by `ssn_last_four` returns correct results
- Integration: PATCH merges JSONB `party_data` without overwriting unmodified fields

#### 2.3 — Loan Product Catalogue

**What**: Implement loan product definitions with JSONB-based product configuration and field schema validation.

**Design**:

```python
# src/models/application.py (partial)
class LoanProduct(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "loan_product"
    product_code: sa.orm.Mapped[str] = sa.orm.mapped_column(
        sa.String(30), unique=True, nullable=False
    )
    product_name: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(100), nullable=False)
    product_category: sa.orm.Mapped[str] = sa.orm.mapped_column(
        sa.String(30), nullable=False  # mortgage, consumer, commercial, sba, heloc
    )
    min_amount: sa.orm.Mapped[Decimal | None] = sa.orm.mapped_column(sa.Numeric(14, 2))
    max_amount: sa.orm.Mapped[Decimal | None] = sa.orm.mapped_column(sa.Numeric(14, 2))
    min_term_months: sa.orm.Mapped[int | None] = sa.orm.mapped_column(sa.Integer)
    max_term_months: sa.orm.Mapped[int | None] = sa.orm.mapped_column(sa.Integer)
    product_config: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
    field_schema: sa.orm.Mapped[dict | None] = sa.orm.mapped_column(sa.JSON)
    is_active: sa.orm.Mapped[bool] = sa.orm.mapped_column(default=True)
```

Seed data includes at least:
- `CONV_30_FIXED` — Conventional 30-year fixed mortgage
- `CONV_15_FIXED` — Conventional 15-year fixed mortgage
- `FHA_30_FIXED` — FHA 30-year fixed
- `AUTO_NEW` — New vehicle auto loan
- `AUTO_USED` — Used vehicle auto loan
- `PERSONAL_UNSECURED` — Personal unsecured consumer loan
- `SBA_7A` — SBA 7(a) small business loan
- `COMMERCIAL_CRE` — Commercial real estate
- `HELOC` — Home equity line of credit

API endpoints:
- `POST /api/v1/products` — Create product (admin only)
- `GET /api/v1/products` — List active products
- `GET /api/v1/products/{id}` — Get product detail including config and field schema
- `PATCH /api/v1/products/{id}` — Update product

**Testing**:
- Unit: `LoanProduct` validates `product_category` enum values
- Integration: Create mortgage product with full `product_config` JSONB, retrieve and verify
- Integration: List products returns only active products by default
- Integration: Seed migration creates all default products
- Fixture: `fixtures/products/` contains JSON files for each default product

#### 2.4 — Loan Application Model and CRUD

**What**: Implement the loan application entity with JSONB fields for product-specific data and compliance data.

**Design**:

```python
# src/models/application.py (continued)
class LoanApplication(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "loan_application"
    application_number: sa.orm.Mapped[str] = sa.orm.mapped_column(
        sa.String(30), unique=True, nullable=False
    )
    loan_product_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_product.id"), nullable=False
    )
    status: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), default="draft")
    channel: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), nullable=False)
    requested_amount: sa.orm.Mapped[Decimal] = sa.orm.mapped_column(
        sa.Numeric(14, 2), nullable=False
    )
    approved_amount: sa.orm.Mapped[Decimal | None] = sa.orm.mapped_column(sa.Numeric(14, 2))
    loan_term_months: sa.orm.Mapped[int] = sa.orm.mapped_column(sa.Integer, nullable=False)
    loan_purpose: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(50), nullable=False)
    loan_officer_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("staff.id"))
    processor_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("staff.id"))
    underwriter_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("staff.id"))
    branch_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("branch.id"))
    submitted_at: sa.orm.Mapped[datetime | None] = sa.orm.mapped_column(sa.DateTime(timezone=True))
    decision_at: sa.orm.Mapped[datetime | None] = sa.orm.mapped_column(sa.DateTime(timezone=True))
    funded_at: sa.orm.Mapped[datetime | None] = sa.orm.mapped_column(sa.DateTime(timezone=True))
    closed_at: sa.orm.Mapped[datetime | None] = sa.orm.mapped_column(sa.DateTime(timezone=True))
    application_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
    compliance_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)

class ApplicationParty(Base, UUIDMixin):
    __tablename__ = "application_party"
    application_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id"), nullable=False
    )
    party_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("party.id"), nullable=False
    )
    role: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), nullable=False)
    role_order: sa.orm.Mapped[int] = sa.orm.mapped_column(default=1)
    created_at: sa.orm.Mapped[datetime] = sa.orm.mapped_column(
        sa.DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )
    __table_args__ = (
        sa.UniqueConstraint("application_id", "party_id", "role"),
    )
```

Application number format: `LN-{YYYY}-{sequence:05d}` (e.g., `LN-2026-00042`). Generated via a PostgreSQL sequence.

Status state machine:
```
draft -> submitted -> processing -> underwriting ->
  approved / conditionally_approved / denied / withdrawn
approved -> funded -> closed
```

Validation at creation:
- `requested_amount` must be within the product's `min_amount`/`max_amount`
- `loan_term_months` must be within the product's `min_term_months`/`max_term_months`
- `channel` must be one of: web, mobile, branch, call_center, broker
- `loan_purpose` must be valid for the product category

API endpoints:
- `POST /api/v1/applications` — Create application (includes borrower party IDs)
- `GET /api/v1/applications` — List applications (paginated, filterable by status, officer, date range)
- `GET /api/v1/applications/{id}` — Get application with parties, product detail
- `PATCH /api/v1/applications/{id}` — Update application fields
- `POST /api/v1/applications/{id}/status` — Transition status (validates state machine)
- `POST /api/v1/applications/{id}/parties` — Add party with role
- `DELETE /api/v1/applications/{id}/parties/{party_id}` — Remove party from application

**Testing**:
- Unit: Application number generates correct format `LN-2026-XXXXX`
- Unit: Status transition validates allowed transitions (draft->submitted OK, draft->funded rejected)
- Unit: `requested_amount` outside product range rejected with validation error
- Integration: Create application with borrower and co-borrower parties, retrieve with full party details
- Integration: Status transition from `draft` to `submitted` sets `submitted_at` timestamp
- Integration: Filter applications by status returns only matching records
- Integration: Adding duplicate party+role to application returns 409
- Fixture: `sample_application.json` with mortgage, consumer, and commercial examples

#### 2.5 — Audit Log Infrastructure

**What**: Implement the audit log model and a reusable service that records all data changes.

**Design**:

```python
# src/models/audit.py
class AuditLog(Base, UUIDMixin):
    __tablename__ = "audit_log"
    entity_type: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(50), nullable=False)
    entity_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(sa.Uuid, nullable=False)
    action: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), nullable=False)
    changes: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
    performed_by_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.Uuid)
    performed_by_type: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), nullable=False)
    ip_address: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(45))
    performed_at: sa.orm.Mapped[datetime] = sa.orm.mapped_column(
        sa.DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )
    __table_args__ = (
        sa.Index("idx_audit_entity", "entity_type", "entity_id"),
        sa.Index("idx_audit_time", "performed_at"),
    )

# src/services/audit_service.py
class AuditService:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def log(
        self,
        entity_type: str,
        entity_id: UUID,
        action: str,  # create, update, delete, view
        changes: dict,
        performed_by_id: UUID | None = None,
        performed_by_type: str = "system",
        ip_address: str | None = None,
    ) -> AuditLog: ...

    async def get_history(
        self,
        entity_type: str,
        entity_id: UUID,
        limit: int = 50,
        offset: int = 0,
    ) -> list[AuditLog]: ...
```

The `changes` JSONB stores:
```json
{
  "fields_changed": ["status", "approved_amount"],
  "before": {"status": "underwriting", "approved_amount": null},
  "after": {"status": "approved", "approved_amount": 340000.00}
}
```

API endpoint:
- `GET /api/v1/audit/{entity_type}/{entity_id}` — Get audit history for an entity

**Testing**:
- Unit: `AuditService.log` creates audit record with correct before/after diff
- Integration: Creating a loan application produces a `create` audit log entry
- Integration: Updating application status produces an `update` audit log with before/after
- Integration: Audit history endpoint returns records in reverse chronological order

---

## Phase 3: Authentication, Authorization & Security

### Purpose
Implement JWT-based authentication, role-based access control (RBAC), and security middleware. After this phase, all API endpoints require authentication, staff members can log in, and actions are restricted based on role permissions.

### Tasks

#### 3.1 — Authentication System

**What**: Implement JWT token issuance, validation, and refresh for staff users.

**Design**:

```python
# src/api/v1/auth.py
# POST /api/v1/auth/login
# Request: { "email": "user@bank.com", "password": "..." }
# Response: { "access_token": "...", "token_type": "bearer", "expires_in": 28800 }

# POST /api/v1/auth/refresh
# Header: Authorization: Bearer <refresh_token>
# Response: { "access_token": "...", "token_type": "bearer", "expires_in": 28800 }

# POST /api/v1/auth/logout
# Invalidates the current token (adds to Redis blocklist)

# src/api/deps.py
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_session),
) -> Staff:
    """Decode JWT, validate against blocklist, return Staff or raise 401."""
    ...

async def require_role(*roles: str):
    """Dependency factory: raises 403 if current user's role not in allowed roles."""
    ...
```

JWT payload:
```json
{
  "sub": "<staff_uuid>",
  "email": "user@bank.com",
  "role": "loan_officer",
  "org_id": "<org_uuid>",
  "branch_id": "<branch_uuid>",
  "permissions": ["view_applications", "edit_applications"],
  "exp": 1748500000,
  "iat": 1748471200
}
```

Password hashing uses `passlib` with bcrypt, minimum 12 rounds.

**Testing**:
- Unit: JWT encodes and decodes correctly with expected claims
- Unit: Expired JWT raises 401
- Unit: Invalid JWT signature raises 401
- Integration: Login with valid credentials returns JWT token
- Integration: Login with wrong password returns 401
- Integration: Accessing protected endpoint without token returns 401
- Integration: Logged-out token (in blocklist) returns 401

#### 3.2 — Role-Based Access Control (RBAC)

**What**: Enforce permission checks on all API endpoints based on staff role.

**Design**:

Permission matrix:

| Permission | admin | loan_officer | processor | underwriter | closer | compliance_officer |
|------------|-------|-------------|-----------|-------------|--------|-------------------|
| manage_org | yes | | | | | |
| manage_staff | yes | | | | | |
| manage_products | yes | | | | | |
| create_application | yes | yes | | | | |
| view_applications | yes | yes | yes | yes | yes | yes |
| edit_application | yes | yes | yes | | | |
| assign_application | yes | yes | | | | |
| pull_credit | yes | | yes | yes | | |
| underwrite | yes | | | yes | | |
| override_decision | yes | | | yes | | |
| manage_documents | yes | yes | yes | yes | yes | |
| manage_compliance | yes | | | | | yes |
| view_audit | yes | | | | | yes |
| fund_loan | yes | | | | yes | |
| view_reports | yes | yes | | | | yes |

```python
# src/services/rbac.py
ROLE_PERMISSIONS: dict[str, set[str]] = {
    "admin": {"manage_org", "manage_staff", "manage_products", ...},
    "loan_officer": {"create_application", "view_applications", ...},
    ...
}

def check_permission(user: Staff, permission: str) -> bool:
    role_perms = ROLE_PERMISSIONS.get(user.role, set())
    user_perms = set(user.permissions or [])
    return permission in (role_perms | user_perms)
```

Apply as FastAPI dependency:
```python
@router.post("/applications")
async def create_application(
    data: ApplicationCreate,
    user: Staff = Depends(require_permission("create_application")),
    session: AsyncSession = Depends(get_session),
): ...
```

**Testing**:
- Unit: Admin role has all permissions
- Unit: Loan officer cannot `underwrite` or `fund_loan`
- Unit: Custom user permissions override role defaults
- Integration: Loan officer creating application succeeds (201)
- Integration: Processor creating application returns 403
- Integration: Underwriter accessing compliance endpoints returns 403

#### 3.3 — API Security Middleware

**What**: Add rate limiting, request logging, and security headers.

**Design**:

- Rate limiting via Redis: 100 requests/minute per user for general endpoints, 10/minute for credit pull endpoints
- Security headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`, `X-Request-ID` generation
- Request/response logging: log method, path, status, duration, user ID (exclude body for PII safety)
- PII masking in logs: SSN, account numbers, passwords never logged

**Testing**:
- Integration: 101st request within a minute returns 429 Too Many Requests
- Integration: Response includes `X-Request-ID` header
- Integration: Security headers present on all responses
- Unit: PII masking function redacts SSN patterns in log output

---

## Phase 4: Document Management & Storage

### Purpose
Implement document upload, storage, retrieval, versioning, and eSignature tracking. After this phase, staff and borrowers can upload documents to loan applications, download them securely, and track eSignature envelopes.

### Tasks

#### 4.1 — Document Upload and Storage

**What**: Implement S3-compatible document storage with presigned URLs for secure upload and download.

**Design**:

```python
# src/models/document.py
class Document(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "document"
    application_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id"), nullable=False
    )
    party_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("party.id"))
    document_type: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(50), nullable=False)
    document_category: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), nullable=False)
    file_name: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(255), nullable=False)
    file_size_bytes: sa.orm.Mapped[int | None] = sa.orm.mapped_column(sa.BigInteger)
    mime_type: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(100))
    storage_path: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(500), nullable=False)
    status: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), default="uploaded")
    version: sa.orm.Mapped[int] = sa.orm.mapped_column(default=1)
    supersedes_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("document.id"))
    extraction_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)

# src/integrations/storage.py
class StorageService:
    async def generate_upload_url(
        self, application_id: UUID, file_name: str, content_type: str
    ) -> tuple[str, str]:
        """Returns (presigned_upload_url, storage_path)."""
        ...

    async def generate_download_url(self, storage_path: str, expires_in: int = 3600) -> str:
        """Returns presigned download URL."""
        ...

    async def delete_object(self, storage_path: str) -> None: ...
```

Storage path format: `documents/{org_id}/{application_number}/{document_type}/{uuid}.{ext}`

Document types: `pay_stub`, `w2`, `tax_return`, `bank_statement`, `appraisal`, `title_report`, `insurance`, `loan_estimate`, `closing_disclosure`, `note`, `deed`, `hud1`, `dl_copy`, `other`

Document categories: `income`, `asset`, `credit`, `compliance`, `closing`, `servicing`, `identity`

API endpoints:
- `POST /api/v1/applications/{app_id}/documents/upload-url` — Get presigned upload URL
- `POST /api/v1/applications/{app_id}/documents` — Register uploaded document
- `GET /api/v1/applications/{app_id}/documents` — List documents (filterable by type, category, status)
- `GET /api/v1/documents/{id}` — Get document metadata
- `GET /api/v1/documents/{id}/download` — Get presigned download URL
- `POST /api/v1/documents/{id}/verify` — Mark document as verified
- `POST /api/v1/documents/{id}/reject` — Mark document as rejected
- `POST /api/v1/documents/{id}/new-version` — Upload new version (sets supersedes_id)

**Testing**:
- Integration (mocked S3): Upload URL generation returns valid presigned URL
- Integration (mocked S3): Download URL generation returns valid presigned URL with expiry
- Integration: Registering document creates record with correct storage path
- Integration: Document versioning sets `supersedes_id` and increments version
- Integration: List documents filters by type and category correctly
- Unit: Storage path generation follows correct format

#### 4.2 — eSignature Envelope Tracking

**What**: Track eSignature envelopes and recipient status for DocuSign integration.

**Design**:

```python
# src/models/document.py (continued)
class EsignatureEnvelope(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "esignature_envelope"
    application_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id"), nullable=False
    )
    provider: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), nullable=False)
    external_envelope_id: sa.orm.Mapped[str] = sa.orm.mapped_column(
        sa.String(100), nullable=False
    )
    status: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), nullable=False)
    envelope_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
```

API endpoints:
- `POST /api/v1/applications/{app_id}/esignatures` — Create envelope (sends via DocuSign API)
- `GET /api/v1/applications/{app_id}/esignatures` — List envelopes
- `POST /api/v1/webhooks/docusign` — Webhook handler for DocuSign status updates

**Testing**:
- Integration (mocked DocuSign): Create envelope returns envelope ID
- Integration: Webhook updates envelope and recipient status correctly
- Integration: Webhook with invalid signature returns 401
- Unit: Envelope status transitions validated (created->sent->delivered->signed)

---

## Phase 5: Workflow Engine & Task Management

### Purpose
Implement configurable workflow templates, workflow instances that drive loan applications through processing stages, and task assignment with SLA tracking. After this phase, submitting an application automatically creates a workflow with tasks assigned to the appropriate staff roles.

### Tasks

#### 5.1 — Workflow Template System

**What**: Implement workflow templates that define the processing steps for each loan product category.

**Design**:

```python
# src/models/workflow.py
class WorkflowTemplate(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "workflow_template"
    name: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(100), nullable=False)
    product_category: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), nullable=False)
    workflow_definition: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, nullable=False)
    is_active: sa.orm.Mapped[bool] = sa.orm.mapped_column(default=True)
```

Default workflow templates seeded for mortgage, consumer, and commercial categories.

Mortgage workflow definition example:
```json
{
  "steps": [
    {"name": "Application Review", "order": 1, "role": "processor", "sla_hours": 24, "required": true},
    {"name": "Credit Pull", "order": 2, "role": "processor", "sla_hours": 4, "required": true},
    {"name": "Document Collection", "order": 3, "role": "processor", "sla_hours": 72, "required": true},
    {"name": "Income Verification", "order": 4, "role": "processor", "sla_hours": 48, "required": true},
    {"name": "Underwriting Review", "order": 5, "role": "underwriter", "sla_hours": 48, "required": true},
    {"name": "Condition Clearing", "order": 6, "role": "processor", "sla_hours": 72, "required": false},
    {"name": "Final Approval", "order": 7, "role": "underwriter", "sla_hours": 24, "required": true},
    {"name": "Closing Preparation", "order": 8, "role": "closer", "sla_hours": 48, "required": true}
  ]
}
```

**Testing**:
- Integration: Seed migration creates default workflow templates
- Integration: Create custom workflow template via API
- Unit: Workflow definition validates step ordering and required role fields

#### 5.2 — Task Engine and SLA Tracking

**What**: Implement task creation, assignment, completion, and SLA monitoring tied to workflow steps.

**Design**:

```python
# src/models/workflow.py (continued)
class Task(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "task"
    application_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id"), nullable=False
    )
    task_type: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(50), nullable=False)
    title: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(255), nullable=False)
    description: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.Text)
    assigned_to_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("staff.id"))
    assigned_role: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(30))
    priority: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(10), default="normal")
    status: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), default="pending")
    due_at: sa.orm.Mapped[datetime | None] = sa.orm.mapped_column(sa.DateTime(timezone=True))
    completed_at: sa.orm.Mapped[datetime | None] = sa.orm.mapped_column(sa.DateTime(timezone=True))
    completed_by_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("staff.id"))
    task_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)

# src/services/workflow_service.py
class WorkflowService:
    async def start_workflow(
        self, application_id: UUID, product_category: str
    ) -> list[Task]:
        """Creates tasks from the workflow template for the given product category.
        Sets due_at based on SLA hours. Auto-assigns based on application's
        loan_officer_id / processor_id if set."""
        ...

    async def complete_task(
        self, task_id: UUID, completed_by: UUID, notes: dict | None = None
    ) -> Task:
        """Marks task complete, checks if next step should be activated,
        updates application status if all required tasks for current stage are done."""
        ...

    async def get_overdue_tasks(self) -> list[Task]:
        """Returns tasks past their SLA deadline."""
        ...

    async def reassign_task(self, task_id: UUID, new_assignee_id: UUID) -> Task: ...
```

When an application transitions from `draft` to `submitted`, `WorkflowService.start_workflow` is called automatically.

API endpoints:
- `GET /api/v1/tasks` — List tasks for current user (filterable by status, priority, overdue)
- `GET /api/v1/applications/{app_id}/tasks` — List tasks for an application
- `PATCH /api/v1/tasks/{id}` — Update task (assign, change priority)
- `POST /api/v1/tasks/{id}/complete` — Complete task
- `POST /api/v1/tasks/{id}/reassign` — Reassign task

**Testing**:
- Integration: Submitting application creates workflow tasks per template
- Integration: Task `due_at` computed correctly from SLA hours
- Integration: Completing all required tasks for a stage advances application status
- Integration: Overdue task query returns tasks past SLA deadline
- Unit: Task reassignment preserves audit trail
- Unit: Completing task sets `completed_at` and `completed_by_id`

#### 5.3 — Notification Service

**What**: Send notifications (email, in-app) when tasks are assigned, SLA approaching, or application status changes.

**Design**:

```python
# src/models/notification.py (within workflow.py or separate)
class Notification(Base, UUIDMixin):
    __tablename__ = "notification"
    application_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id")
    )
    recipient_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.Uuid)
    recipient_type: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(10))
    channel: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), nullable=False)
    template_code: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(50), nullable=False)
    status: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), default="pending")
    notification_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
    created_at: sa.orm.Mapped[datetime] = sa.orm.mapped_column(
        sa.DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )

# src/services/notification_service.py
class NotificationService:
    async def send(
        self,
        recipient_id: UUID,
        recipient_type: str,  # staff, party
        channel: str,  # email, in_app
        template_code: str,
        context: dict,
        application_id: UUID | None = None,
    ) -> Notification: ...
```

Notification template codes:
- `task_assigned` — New task assigned to staff
- `task_overdue` — Task past SLA deadline
- `status_changed` — Application status update (to borrower and staff)
- `document_required` — Request for additional documents (to borrower)
- `document_uploaded` — Borrower uploaded a document (to processor)
- `decision_rendered` — Underwriting decision made (to loan officer)

Celery task handles async delivery. For MVP, email via SMTP and in-app via database records polled by the frontend.

**Testing**:
- Integration: Assigning task creates `task_assigned` notification for assignee
- Integration: Notification status transitions from pending to sent to delivered
- Unit: Template rendering produces correct email subject and body
- Unit (mocked SMTP): Email notification delivery succeeds

---

## Phase 6: Credit & Underwriting Engine

### Purpose
Implement credit bureau integration, the automated decisioning engine, underwriting conditions, and adverse action notice generation. This is the core value of the LOS. After this phase, applications can be automatically scored, decisions rendered, conditions assigned, and adverse action notices generated for denials.

### Tasks

#### 6.1 — Credit Bureau Integration

**What**: Build the integration layer for pulling credit reports from Equifax, Experian, and TransUnion.

**Design**:

```python
# src/integrations/credit_bureau.py
class CreditBureauClient:
    """Abstract interface for credit bureau connections."""

    async def pull_credit(
        self,
        party: Party,
        bureau: str,  # equifax, experian, transunion
        report_type: str,  # individual, joint, tri_merge
        permissible_purpose: str,  # FCRA purpose code
    ) -> CreditReportResult: ...

class CreditReportResult(BaseModel):
    bureau: str
    credit_score: int
    score_model: str
    report_reference: str
    expiration_date: date
    summary: dict  # total_accounts, open_accounts, collections, etc.
    tradelines: list[dict]
    inquiries: list[dict]
    public_records: list[dict]
    raw_data_encrypted: bytes  # Full report encrypted with AES-256

# src/models/credit.py
class CreditReport(Base, UUIDMixin):
    __tablename__ = "credit_report"
    party_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(sa.ForeignKey("party.id"), nullable=False)
    application_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id"), nullable=False
    )
    bureau: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), nullable=False)
    report_type: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), nullable=False)
    permissible_purpose: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(50), nullable=False)
    pull_date: sa.orm.Mapped[datetime] = sa.orm.mapped_column(sa.DateTime(timezone=True), nullable=False)
    expiration_date: sa.orm.Mapped[date | None] = sa.orm.mapped_column(sa.Date)
    credit_score: sa.orm.Mapped[int | None] = sa.orm.mapped_column(sa.Integer)
    score_model: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(30))
    report_summary: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
    report_data_encrypted: sa.orm.Mapped[bytes | None] = sa.orm.mapped_column(sa.LargeBinary)
    created_at: sa.orm.Mapped[datetime] = sa.orm.mapped_column(
        sa.DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )
```

Rate limiting: Max 1 credit pull per party per bureau per 24 hours (FCRA compliance).

Credit pull is an async Celery task:
```python
# src/tasks/credit_tasks.py
@celery_app.task
async def pull_credit_report(
    application_id: str, party_id: str, bureau: str, report_type: str
) -> dict: ...
```

API endpoints:
- `POST /api/v1/applications/{app_id}/credit/pull` — Initiate credit pull (async, returns task ID)
- `GET /api/v1/applications/{app_id}/credit` — List credit reports for application
- `GET /api/v1/credit/{id}` — Get credit report detail (score, summary; encrypted raw data excluded)

**Testing**:
- Unit (mocked bureau): Tri-merge pull returns scores from all three bureaus
- Unit: FCRA rate limiting blocks duplicate pull within 24 hours
- Unit: Credit report data is encrypted before storage
- Integration (mocked bureau): Full credit pull flow: API call -> Celery task -> stored report
- Integration: Expired credit report flagged when accessed after expiration date
- Fixture: `sample_credit_report.json` with representative data

#### 6.2 — Automated Decisioning Engine

**What**: Build the configurable rules-based and AI-assisted decisioning engine that evaluates applications and renders approve/deny/refer decisions.

**Design**:

```python
# src/engine/decision_engine.py
class DecisionEngine:
    """Evaluates a loan application against credit policy rules and renders a decision."""

    async def evaluate(
        self,
        application: LoanApplication,
        credit_reports: list[CreditReport],
        product: LoanProduct,
    ) -> DecisionResult: ...

class DecisionResult(BaseModel):
    decision: str  # approve, deny, refer, conditionally_approve
    risk_grade: str  # A1, A2, B1, B2, C1, C2, D, E
    risk_metrics: RiskMetrics
    conditions: list[ConditionResult]
    adverse_action_reasons: list[AdverseActionReason]
    explainability: ExplainabilityData
    model_id: str
    model_version: str

class RiskMetrics(BaseModel):
    dti_ratio: Decimal
    housing_ratio: Decimal | None
    ltv_ratio: Decimal | None
    representative_credit_score: int
    residual_income: Decimal | None
    total_monthly_debt: Decimal
    total_monthly_income: Decimal

class AdverseActionReason(BaseModel):
    code: str  # e.g., "R001" — standardised adverse action reason codes
    text: str  # ECOA-compliant human-readable reason

class ExplainabilityData(BaseModel):
    top_factors: list[dict]  # {"factor": "credit_score", "impact": "positive", "weight": 0.35}
    natural_language_explanation: str  # LLM-generated for adverse action notices

# src/engine/rules.py
class CreditPolicyRules:
    """Configurable credit policy rules loaded from loan_product.product_config."""

    def evaluate_dti(self, dti: Decimal, max_dti: Decimal) -> RuleResult: ...
    def evaluate_ltv(self, ltv: Decimal, max_ltv: Decimal) -> RuleResult: ...
    def evaluate_credit_score(self, score: int, min_score: int) -> RuleResult: ...
    def evaluate_employment_stability(self, months: int, min_months: int) -> RuleResult: ...
    def evaluate_all(self, metrics: RiskMetrics, policy: dict) -> list[RuleResult]: ...
```

Decision logic:
1. Calculate `RiskMetrics` from application data, party data, and credit reports
2. Evaluate all credit policy rules from product config
3. If all rules pass and score > threshold: `approve`
4. If any hard-fail rules triggered: `deny` (with adverse action reasons)
5. If soft-fail rules triggered: `refer` (for manual underwriting review)
6. Assign risk grade based on composite scoring
7. Generate conditions for conditional approvals

```python
# src/models/underwriting.py
class UnderwritingDecision(Base, UUIDMixin):
    __tablename__ = "underwriting_decision"
    application_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id"), nullable=False
    )
    decision_type: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), nullable=False)
    decision: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), nullable=False)
    decided_by_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("staff.id"))
    decided_at: sa.orm.Mapped[datetime] = sa.orm.mapped_column(
        sa.DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )
    is_current: sa.orm.Mapped[bool] = sa.orm.mapped_column(default=True)
    decision_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
    created_at: sa.orm.Mapped[datetime] = sa.orm.mapped_column(
        sa.DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )
```

API endpoints:
- `POST /api/v1/applications/{app_id}/underwrite` — Run automated decisioning
- `POST /api/v1/applications/{app_id}/underwrite/manual` — Record manual underwriting decision
- `POST /api/v1/applications/{app_id}/underwrite/override` — Override automated decision (senior underwriter)
- `GET /api/v1/applications/{app_id}/decisions` — List all decisions (history)
- `GET /api/v1/decisions/{id}` — Get decision detail with risk metrics and explainability

**Testing**:
- Unit: DTI ratio above max threshold produces `deny` with correct adverse action reason
- Unit: Credit score below minimum produces `deny` with reason code `R001`
- Unit: All rules passing produces `approve` with risk grade
- Unit: Some soft-fail rules produce `refer` for manual review
- Unit: Risk grade calculation assigns correct grade for various score/DTI/LTV combinations
- Integration: Full automated decisioning flow for mortgage application
- Integration: Manual decision override marks previous decision as `is_current=false`
- Integration: Decision creates audit log entry
- Fixture: Test cases with known inputs and expected decisions for each loan product

#### 6.3 — Underwriting Conditions

**What**: Manage stipulations/conditions attached to conditional approvals.

**Design**:

Conditions are stored within the `decision_data` JSONB of `UnderwritingDecision`:

```json
{
  "conditions": [
    {
      "id": "uuid",
      "type": "prior_to_closing",
      "category": "income",
      "description": "Provide most recent 30 days of pay stubs",
      "status": "outstanding",
      "assigned_to_id": null,
      "due_date": "2026-06-15",
      "cleared_by_id": null,
      "cleared_at": null
    }
  ]
}
```

Service methods:
```python
# src/services/underwriting_service.py
class UnderwritingService:
    async def add_condition(self, decision_id: UUID, condition: ConditionData) -> dict: ...
    async def clear_condition(self, decision_id: UUID, condition_id: str, cleared_by: UUID) -> dict: ...
    async def waive_condition(self, decision_id: UUID, condition_id: str, waived_by: UUID) -> dict: ...
    async def get_outstanding_conditions(self, application_id: UUID) -> list[dict]: ...
```

API endpoints:
- `POST /api/v1/decisions/{id}/conditions` — Add condition
- `PATCH /api/v1/decisions/{id}/conditions/{cond_id}` — Update condition status
- `POST /api/v1/decisions/{id}/conditions/{cond_id}/clear` — Clear condition
- `GET /api/v1/applications/{app_id}/conditions` — Get all outstanding conditions

**Testing**:
- Integration: Adding condition updates decision JSONB
- Integration: Clearing all conditions on a `conditionally_approved` decision triggers notification
- Unit: Condition status transitions validated (outstanding->received->reviewed->cleared)
- Unit: Only underwriters can clear conditions (RBAC enforced)

#### 6.4 — Adverse Action Notice Generation (ECOA Compliance)

**What**: Generate ECOA-compliant adverse action notices with plain-language explanations when an application is denied.

**Design**:

```python
# src/engine/explainability.py
class AdverseActionGenerator:
    """Generates ECOA-compliant adverse action notices using LLM for natural-language explanations."""

    async def generate_notice(
        self,
        decision: DecisionResult,
        application: LoanApplication,
        borrower: Party,
    ) -> AdverseActionNotice: ...

class AdverseActionNotice(BaseModel):
    reason_codes: list[str]
    reason_texts: list[str]  # Standardised adverse action reason texts
    natural_language_explanation: str  # LLM-generated, borrower-friendly explanation
    credit_score_used: int
    score_model: str
    score_range: str  # e.g., "300-850"
    credit_bureau: str
    notice_date: date
    response_deadline: date  # 60 days per ECOA
```

LLM prompt template for natural-language explanation:
```
You are a compliance assistant generating adverse action notices for loan applicants.
Given the following denial reasons and application data, write a clear, professional,
borrower-friendly explanation of why the application was not approved.

Requirements:
- Use plain English (no jargon)
- Reference specific factors from the denial reasons
- Do not disclose internal scoring models or thresholds
- Be factual and empathetic
- Include guidance on how the applicant might improve their profile
- Must comply with ECOA Regulation B requirements

Denial reasons: {reasons}
Application type: {product_name}
Requested amount: {amount}
```

PDF generation using Jinja2 template + reportlab for the formal notice document.

**Testing**:
- Unit (mocked LLM): Notice generation produces all required fields
- Unit: All standardised reason codes map to valid ECOA reason texts
- Unit: Notice includes credit score disclosure per FCRA requirements
- Integration: Denial decision triggers adverse action notice generation
- Integration: Generated PDF contains all required regulatory content
- Fixture: Expected notice outputs for common denial scenarios

---

## Phase 7: Compliance & Regulatory Reporting

### Purpose
Implement TRID disclosure tracking, HMDA data capture and reporting, and fee tolerance monitoring. After this phase, the system satisfies core regulatory requirements for US mortgage lending.

### Tasks

#### 7.1 — TRID Disclosure Management

**What**: Track Loan Estimate (LE) and Closing Disclosure (CD) issuance, receipt, and waiting period compliance per TILA-RESPA Integrated Disclosure rules.

**Design**:

TRID rules implemented:
- LE must be issued within 3 business days of application
- CD must be issued at least 3 business days before closing
- Fee tolerance tracking: zero-tolerance, 10% tolerance, and unlimited categories
- Revised LE/CD tracking with valid change-of-circumstance reasons

```python
# src/services/compliance_service.py
class TRIDService:
    async def issue_loan_estimate(
        self, application_id: UUID, fees: list[FeeData]
    ) -> TRIDDisclosure: ...

    async def issue_closing_disclosure(
        self, application_id: UUID, final_fees: list[FeeData]
    ) -> TRIDDisclosure: ...

    async def check_tolerance_violations(
        self, application_id: UUID
    ) -> list[ToleranceViolation]: ...

    async def calculate_waiting_period_end(
        self, issued_date: date, received_date: date | None
    ) -> date:
        """Returns the earliest permissible closing date.
        3 business days after borrower receives the disclosure.
        If received date is None, assumes receipt 3 days after mailing."""
        ...

    def categorise_fee_tolerance(self, fee_type: str) -> str:
        """Returns 'zero', 'ten_percent', or 'unlimited' per TRID rules."""
        ...
```

Tolerance categories per TRID:
- **Zero tolerance**: transfer taxes, fees paid to lender (origination charges)
- **10% tolerance**: third-party fees where lender selects provider (appraisal, credit report, title search)
- **Unlimited**: third-party fees where borrower selects provider, prepaid interest, insurance

```python
# src/models/compliance.py
class TRIDDisclosure(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "trid_disclosure"
    application_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id"), nullable=False
    )
    disclosure_type: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), nullable=False)
    version_number: sa.orm.Mapped[int] = sa.orm.mapped_column(default=1)
    issued_date: sa.orm.Mapped[date] = sa.orm.mapped_column(sa.Date, nullable=False)
    received_date: sa.orm.Mapped[date | None] = sa.orm.mapped_column(sa.Date)
    waiting_period_end: sa.orm.Mapped[date | None] = sa.orm.mapped_column(sa.Date)
    is_compliant: sa.orm.Mapped[bool | None] = sa.orm.mapped_column(sa.Boolean)
    compliance_notes: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.Text)
    fee_snapshot: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
    document_id: sa.orm.Mapped[uuid4 | None] = sa.orm.mapped_column(sa.ForeignKey("document.id"))

class LoanFee(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "loan_fee"
    application_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id"), nullable=False
    )
    fee_type: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(50), nullable=False)
    fee_name: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(100), nullable=False)
    amount: sa.orm.Mapped[Decimal] = sa.orm.mapped_column(sa.Numeric(12, 2), nullable=False)
    paid_by: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(20), nullable=False)
    tolerance_category: sa.orm.Mapped[str | None] = sa.orm.mapped_column(sa.String(20))
    fee_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)
```

API endpoints:
- `POST /api/v1/applications/{app_id}/compliance/trid/loan-estimate` — Issue LE
- `POST /api/v1/applications/{app_id}/compliance/trid/closing-disclosure` — Issue CD
- `GET /api/v1/applications/{app_id}/compliance/trid` — Get TRID status and timeline
- `GET /api/v1/applications/{app_id}/compliance/trid/tolerance` — Check tolerance violations
- `POST /api/v1/applications/{app_id}/fees` — Add fee
- `GET /api/v1/applications/{app_id}/fees` — List fees with tolerance categories

**Testing**:
- Unit: Waiting period calculation handles weekends and federal holidays
- Unit: Fee tolerance categorisation matches TRID rules for all fee types
- Unit: 10% tolerance violation detected when aggregate 10%-category fees exceed LE by more than 10%
- Integration: Issuing LE creates disclosure record with fee snapshot
- Integration: CD cannot be issued if waiting period has not elapsed
- Integration: Revised LE increments version number and tracks change reason
- Fixture: Fee scenarios with known tolerance outcomes

#### 7.2 — HMDA Data Capture and Reporting

**What**: Capture all 110 HMDA-required data fields and generate the annual Loan Activity Register (LAR) file for FFIEC submission.

**Design**:

HMDA data is stored in the `compliance_data` JSONB column on `loan_application`:
```json
{
  "hmda_reportable": true,
  "hmda": {
    "lei": "5493001KJTIIGC8Y1R12",
    "uli": "5493001KJTIIGC8Y1R1220260520001",
    "action_taken": 1,
    "action_taken_date": "2026-05-20",
    "loan_type": 1,
    "loan_purpose": 1,
    "preapproval": 2,
    "construction_method": 1,
    "occupancy_type": 1,
    "loan_amount": 350000.00,
    ...
  }
}
```

```python
# src/services/compliance_service.py
class HMDAService:
    async def capture_hmda_data(
        self, application_id: UUID
    ) -> dict:
        """Auto-populates HMDA fields from application, party, and collateral data.
        Fields that cannot be auto-populated are flagged for manual completion."""
        ...

    async def generate_lar(
        self, organisation_id: UUID, reporting_year: int
    ) -> bytes:
        """Generates the pipe-delimited LAR file for FFIEC submission.
        Format per FFIEC Filing Instructions Guide (FIG)."""
        ...

    async def validate_lar_record(
        self, hmda_data: dict
    ) -> list[HMDAValidationError]:
        """Validates a single LAR record against FFIEC edit checks.
        Returns list of syntactical, validity, and quality edit failures."""
        ...
```

ULI (Universal Loan Identifier) format: `{LEI}{YYYY}{sequence:06d}{check_digit:02d}`

API endpoints:
- `POST /api/v1/applications/{app_id}/compliance/hmda/capture` — Auto-populate HMDA data
- `PATCH /api/v1/applications/{app_id}/compliance/hmda` — Manually update HMDA fields
- `GET /api/v1/applications/{app_id}/compliance/hmda` — Get current HMDA data
- `POST /api/v1/compliance/hmda/validate/{app_id}` — Validate single record
- `GET /api/v1/compliance/hmda/lar/{year}` — Generate LAR file for year (admin/compliance only)

**Testing**:
- Unit: ULI generation produces correct format with valid check digit
- Unit: Auto-population maps application fields to HMDA codes correctly
- Unit: LAR file generation produces pipe-delimited format per FFIEC spec
- Integration: HMDA data captured when mortgage application reaches decision
- Integration: Validation catches missing required fields
- Fixture: Sample HMDA records with known valid/invalid field combinations

---

## Phase 8: AI-Powered Document Intelligence

### Purpose
Implement OCR and LLM-based document extraction, cross-document consistency validation, and AI-assisted income analysis. This is a key AI-native differentiator. After this phase, uploaded pay stubs, tax returns, and bank statements are automatically processed, data extracted, and consistency checked.

### Tasks

#### 8.1 — OCR and Document Processing Pipeline

**What**: Build the async document processing pipeline that extracts text from uploaded documents using OCR and prepares them for AI analysis.

**Design**:

```python
# src/tasks/document_tasks.py
@celery_app.task
async def process_document(document_id: str) -> dict:
    """Pipeline:
    1. Download document from S3
    2. If PDF: convert to images using pdf2image
    3. Run OCR via pytesseract (or cloud OCR for production)
    4. Pass extracted text to LLM for structured data extraction
    5. Store extraction results in document.extraction_data
    6. Update document status to 'processed'
    """
    ...

# src/integrations/document_ai.py
class DocumentAIService:
    async def extract_structured_data(
        self, document_text: str, document_type: str
    ) -> ExtractionResult: ...

class ExtractionResult(BaseModel):
    confidence: float  # 0.0-1.0
    extracted_fields: dict  # Document-type-specific fields
    validation: ValidationResult
```

LLM prompt templates per document type:

**Pay stub extraction prompt**:
```
Extract the following fields from this pay stub text. Return as JSON.
Fields: employer_name, employee_name, pay_period_start, pay_period_end,
        gross_pay, net_pay, ytd_gross, deductions (itemised), pay_frequency

Text:
{ocr_text}
```

**Tax return extraction prompt (Form 1040)**:
```
Extract the following fields from this tax return. Return as JSON.
Fields: tax_year, filing_status, adjusted_gross_income, total_income,
        taxable_income, wages_salaries (line 1), business_income (Schedule C),
        rental_income (Schedule E), interest_income, dividend_income

Text:
{ocr_text}
```

**Bank statement extraction prompt**:
```
Extract the following fields from this bank statement. Return as JSON.
Fields: institution_name, account_holder, account_type, statement_period_start,
        statement_period_end, beginning_balance, ending_balance,
        total_deposits, total_withdrawals, average_daily_balance,
        large_deposits (>$1000, with dates and amounts)

Text:
{ocr_text}
```

**Testing**:
- Unit (mocked LLM): Pay stub extraction returns all expected fields
- Unit (mocked LLM): Tax return extraction handles Schedule C self-employment income
- Unit (mocked LLM): Bank statement extraction identifies large deposits
- Integration (mocked): Full pipeline: upload PDF -> OCR -> LLM extraction -> stored results
- Unit: Low-confidence extractions (<0.80) flagged for manual review
- Fixture: `sample_documents/` directory with test PDFs for each document type

#### 8.2 — Cross-Document Consistency Validation

**What**: Compare extracted data across multiple documents to detect inconsistencies (e.g., name mismatch between pay stub and tax return, income discrepancy between stated and documented).

**Design**:

```python
# src/services/document_service.py
class DocumentValidationService:
    async def validate_consistency(
        self, application_id: UUID
    ) -> ConsistencyReport:
        """Cross-checks extracted data across all documents for an application.

        Checks performed:
        1. Name consistency: borrower name matches across all documents
        2. Employer consistency: employer on pay stub matches employment stated on application
        3. Income consistency: pay stub income * 12 ~ tax return total income (within 15%)
        4. Address consistency: address across documents matches application address
        5. Bank balance: stated assets consistent with bank statement balances
        6. Date relevance: documents are recent enough (pay stub < 60 days, tax return current year)
        """
        ...

class ConsistencyReport(BaseModel):
    overall_status: str  # consistent, inconsistencies_found, insufficient_documents
    checks: list[ConsistencyCheck]

class ConsistencyCheck(BaseModel):
    check_name: str
    status: str  # pass, fail, warning, skipped
    details: str
    documents_compared: list[UUID]
    severity: str  # info, warning, critical
```

**Testing**:
- Unit: Name mismatch between pay stub and application triggers `fail` with `critical` severity
- Unit: 20% income discrepancy between pay stub annualised and tax return triggers `fail`
- Unit: 10% income discrepancy triggers `warning` but not `fail`
- Unit: Missing document for a required check produces `skipped` status
- Integration: Consistency report generated after all required documents are extracted
- Fixture: Test data sets with known consistency/inconsistency patterns

#### 8.3 — AI Income Analysis for Complex Income

**What**: Implement AI-driven income calculation for self-employment, gig work, and rental income using bank statements and tax returns.

**Design**:

```python
# src/engine/income_analysis.py
class IncomeAnalyzer:
    async def analyze_income(
        self,
        party: Party,
        documents: list[Document],
    ) -> IncomeAnalysisResult:
        """Calculates qualifying income from multiple sources.

        For W2 employees: straightforward from pay stubs and W2s
        For self-employed: Schedule C analysis from tax returns + bank statement pattern analysis
        For gig workers: bank statement deposit pattern recognition
        For rental income: Schedule E analysis + lease verification
        """
        ...

class IncomeAnalysisResult(BaseModel):
    total_qualifying_monthly_income: Decimal
    income_sources: list[IncomeSourceAnalysis]
    confidence: float
    methodology_notes: str  # Explanation of how income was calculated

class IncomeSourceAnalysis(BaseModel):
    source_type: str  # W2, self_employment, gig, rental, investment, other
    monthly_amount: Decimal
    calculation_method: str  # 24_month_average, 12_month_average, ytd_annualised
    supporting_documents: list[UUID]
    confidence: float
    notes: str
```

LLM prompt for self-employment income analysis:
```
Analyze the following self-employment income data and calculate qualifying monthly income.
Use standard mortgage underwriting guidelines:
- Use the lower of 2-year average or most recent year if trending down
- Use most recent year if trending up, with documentation of upward trend
- Subtract depreciation add-back per standard guidelines
- Flag any concerns about income stability

Schedule C Data (2 years):
{schedule_c_data}

Bank Statement Summary (12 months):
{bank_statement_summary}
```

**Testing**:
- Unit: W2 income from pay stub correctly annualised for different pay frequencies (weekly, biweekly, semimonthly, monthly)
- Unit: Self-employment income uses 2-year average when income declining
- Unit: Self-employment income uses current year when income trending up
- Unit: Gig income from bank deposits correctly identified and annualised
- Integration (mocked LLM): Full income analysis flow for self-employed borrower
- Fixture: Test cases with known income calculation outcomes

---

## Phase 9: Collateral & Fees

### Purpose
Implement collateral management (real estate, vehicle, equipment) with LTV calculations, appraisal tracking, and the complete fee management system with TRID tolerance monitoring.

### Tasks

#### 9.1 — Collateral Management

**What**: CRUD for collateral records with type-specific JSONB data, LTV/CLTV calculation, and appraisal tracking.

**Design**:

```python
# src/models/collateral.py
class Collateral(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "collateral"
    application_id: sa.orm.Mapped[uuid4] = sa.orm.mapped_column(
        sa.ForeignKey("loan_application.id"), nullable=False
    )
    collateral_type: sa.orm.Mapped[str] = sa.orm.mapped_column(sa.String(30), nullable=False)
    estimated_value: sa.orm.Mapped[Decimal | None] = sa.orm.mapped_column(sa.Numeric(14, 2))
    appraised_value: sa.orm.Mapped[Decimal | None] = sa.orm.mapped_column(sa.Numeric(14, 2))
    ltv_ratio: sa.orm.Mapped[Decimal | None] = sa.orm.mapped_column(sa.Numeric(6, 4))
    cltv_ratio: sa.orm.Mapped[Decimal | None] = sa.orm.mapped_column(sa.Numeric(6, 4))
    lien_position: sa.orm.Mapped[int] = sa.orm.mapped_column(default=1)
    collateral_data: sa.orm.Mapped[dict] = sa.orm.mapped_column(sa.JSON, default=dict)

# src/services/collateral_service.py
class CollateralService:
    async def calculate_ltv(
        self, application_id: UUID
    ) -> Decimal:
        """LTV = loan_amount / appraised_value (or estimated_value if no appraisal)."""
        ...

    async def calculate_cltv(
        self, application_id: UUID
    ) -> Decimal:
        """CLTV considers all liens against the property."""
        ...
```

API endpoints:
- `POST /api/v1/applications/{app_id}/collateral` — Add collateral
- `GET /api/v1/applications/{app_id}/collateral` — List collateral
- `PATCH /api/v1/collateral/{id}` — Update collateral (triggers LTV recalculation)
- `POST /api/v1/collateral/{id}/appraisal` — Record appraisal results

**Testing**:
- Unit: LTV calculated correctly as loan_amount / appraised_value
- Unit: CLTV includes existing liens from party liabilities
- Unit: LTV updates when appraisal value changes
- Integration: Adding real estate collateral with property details in JSONB
- Integration: Vehicle collateral stores VIN, make, model, year correctly

#### 9.2 — Fee Management

**What**: CRUD for loan fees with automatic TRID tolerance categorisation.

**Design**:

API endpoints:
- `POST /api/v1/applications/{app_id}/fees` — Add fee with auto-tolerance categorisation
- `GET /api/v1/applications/{app_id}/fees` — List fees grouped by tolerance category
- `PATCH /api/v1/fees/{id}` — Update fee amount
- `GET /api/v1/applications/{app_id}/fees/tolerance-check` — Compare current fees vs. LE fees

**Testing**:
- Unit: Origination fee categorised as zero-tolerance
- Unit: Appraisal fee categorised as 10%-tolerance when lender selects appraiser
- Unit: Homeowner's insurance categorised as unlimited when borrower selects provider
- Integration: Fee tolerance check detects 10% threshold violation

---

## Phase 10: Analytics Dashboard & Reporting

### Purpose
Build the staff-facing dashboard with pipeline views, performance metrics, and compliance reporting. After this phase, loan officers, managers, and compliance staff have real-time visibility into the loan pipeline.

### Tasks

#### 10.1 — Pipeline Dashboard API

**What**: Build aggregation endpoints for the loan pipeline dashboard.

**Design**:

```python
# src/api/v1/dashboard.py
@router.get("/dashboard/pipeline")
async def get_pipeline(
    branch_id: UUID | None = None,
    loan_officer_id: UUID | None = None,
    date_from: date | None = None,
    date_to: date | None = None,
    user: Staff = Depends(get_current_user),
) -> PipelineDashboard: ...

class PipelineDashboard(BaseModel):
    summary: PipelineSummary
    by_status: list[StatusCount]
    by_product: list[ProductCount]
    by_channel: list[ChannelCount]
    recent_activity: list[ActivityItem]
    overdue_tasks: list[TaskSummary]

class PipelineSummary(BaseModel):
    total_applications: int
    total_volume: Decimal
    avg_days_to_decision: float | None
    avg_days_to_funding: float | None
    approval_rate: float | None
    funded_this_month: int
    funded_volume_this_month: Decimal
```

API endpoints:
- `GET /api/v1/dashboard/pipeline` — Pipeline summary with filters
- `GET /api/v1/dashboard/performance` — Loan officer performance metrics
- `GET /api/v1/dashboard/compliance` — Compliance metrics (TRID timing, HMDA completeness)
- `GET /api/v1/reports/applications` — Application report (CSV/Excel export)
- `GET /api/v1/reports/hmda/{year}` — HMDA LAR report download

**Testing**:
- Integration: Pipeline summary counts match actual application status distribution
- Integration: Date range filter correctly narrows results
- Integration: Loan officer filter shows only their applications
- Integration: CSV export contains correct columns and data

#### 10.2 — Frontend Dashboard Implementation

**What**: Build the Next.js dashboard with pipeline views, application detail pages, and task management.

**Design**:

Key pages:
- `/dashboard` — Pipeline overview with charts (applications by status, volume trends, SLA compliance)
- `/applications` — Searchable, filterable application list
- `/applications/[id]` — Application detail: parties, documents, decisions, conditions, tasks, timeline
- `/applications/new` — New application form (multi-step wizard)
- `/tasks` — Task inbox for current user
- `/parties` — Party directory with search
- `/compliance` — TRID tracker, HMDA status, adverse action notices
- `/reports` — Reporting dashboard with export functionality

Component library (Shadcn/ui based):
- `ApplicationStatusBadge` — Colour-coded status indicator
- `RiskGradeBadge` — Risk grade with colour scale (A=green, E=red)
- `TimelineView` — Chronological audit/event timeline
- `DocumentChecklist` — Required vs. uploaded document status
- `TaskCard` — Task with priority, assignee, SLA countdown
- `MetricCard` — Dashboard KPI with trend indicator

**Testing**:
- E2E (Playwright): Login, navigate to dashboard, verify pipeline chart renders
- E2E: Create new application through wizard, verify it appears in pipeline
- E2E: Upload document, verify it appears in application document list
- E2E: Complete task, verify application status advances
- Component: `ApplicationStatusBadge` renders correct colour for each status
- Component: `TaskCard` shows SLA countdown in red when overdue

---

## Phase 11: Advanced AI Features

### Purpose
Implement the differentiating AI capabilities: real-time fraud detection, predictive pipeline management, and the borrower-facing AI chatbot. These features elevate the platform from a standard LOS to an AI-native system.

### Tasks

#### 11.1 — Fraud and Identity Detection

**What**: Build a real-time fraud scoring system that evaluates applications at submission for synthetic identity fraud and first-party fraud indicators.

**Design**:

```python
# src/engine/fraud_detection.py
class FraudDetector:
    async def evaluate(
        self, application: LoanApplication, parties: list[Party]
    ) -> FraudAssessment: ...

class FraudAssessment(BaseModel):
    overall_risk: str  # low, medium, high, critical
    risk_score: float  # 0.0-1.0
    signals: list[FraudSignal]
    recommendation: str  # proceed, manual_review, block

class FraudSignal(BaseModel):
    signal_type: str  # synthetic_identity, address_mismatch, income_inconsistency, velocity
    severity: str
    description: str
    confidence: float
```

Fraud signals evaluated:
- SSN/name/DOB mismatch patterns (synthetic identity)
- Address consistency across credit report and application
- Application velocity (multiple applications from same device/IP in short timeframe)
- Income-to-stated-income discrepancy beyond threshold
- New credit file (tradeline age < 2 years with high credit request)
- Document tampering indicators from extraction confidence scores

**Testing**:
- Unit: Synthetic identity patterns produce `high` risk score
- Unit: Normal application produces `low` risk score
- Unit: Multiple applications from same IP within 24 hours triggers velocity signal
- Integration: Fraud assessment runs automatically on application submission
- Integration: High-risk assessment blocks automatic approval and creates manual review task

#### 11.2 — Predictive Pipeline Management

**What**: Predict which in-progress loans are at risk of falling out (withdrawal, denial, abandonment) and surface proactive alerts.

**Design**:

```python
# src/engine/pipeline_predictor.py
class PipelinePredictor:
    async def predict_fallout_risk(
        self, application_id: UUID
    ) -> FalloutPrediction: ...

class FalloutPrediction(BaseModel):
    risk_score: float  # 0.0-1.0
    risk_level: str  # low, medium, high
    risk_factors: list[str]  # e.g., "no document upload in 7 days", "rate lock expiring in 3 days"
    recommended_actions: list[str]
```

Risk factors evaluated:
- Days since last borrower activity (document upload, message)
- Outstanding conditions not addressed within SLA
- Rate lock expiration approaching
- Comparable market rates dropped (borrower may shop elsewhere)
- Document collection stalling (missing required docs > 5 days)
- Borrower communication responsiveness declining

API endpoint:
- `GET /api/v1/dashboard/at-risk` — Applications at risk of fallout, sorted by risk score

**Testing**:
- Unit: No document upload for 7 days increases risk score
- Unit: Rate lock expiring in < 3 days raises risk level
- Unit: All documents submitted and conditions clearing reduces risk score
- Integration: At-risk endpoint returns applications sorted by risk

#### 11.3 — Borrower Communication Chatbot

**What**: AI-powered chatbot providing borrowers with application status updates, document guidance, and next-steps information.

**Design**:

```python
# src/services/chatbot_service.py
class BorrowerChatbot:
    async def handle_message(
        self, borrower_id: UUID, application_id: UUID, message: str
    ) -> ChatResponse: ...

class ChatResponse(BaseModel):
    message: str
    suggested_actions: list[SuggestedAction]
    documents_needed: list[str] | None

class SuggestedAction(BaseModel):
    action_type: str  # upload_document, review_disclosure, sign_document
    label: str
    url: str | None
```

Chatbot capabilities:
- Application status inquiry ("Where is my application?")
- Required document guidance ("What documents do I still need?")
- Next steps explanation ("What happens next?")
- Rate lock status ("When does my rate lock expire?")
- Contact routing ("I need to speak with my loan officer")

The chatbot uses the LLM with application context (status, outstanding conditions, documents) injected as system prompt context. PII-aware: borrower can only query their own application.

API endpoint:
- `POST /api/v1/chat/{application_id}` — Send message, receive response (borrower-authenticated)

**Testing**:
- Unit (mocked LLM): Status inquiry returns current application status
- Unit (mocked LLM): Document inquiry lists outstanding required documents
- Unit: Borrower cannot query another borrower's application (auth enforced)
- Integration: Chat history stored for audit trail

---

## Phase 12: MISMO Integration & Data Exchange

### Purpose
Implement MISMO-aligned data export for GSE submission (Fannie Mae DU, Freddie Mac LPA), MISMO XML generation, and the webhook/API ecosystem for third-party integrations.

### Tasks

#### 12.1 — MISMO XML Data Export

**What**: Generate MISMO v3.4-compliant XML for loan data exchange with GSEs and investors.

**Design**:

```python
# src/integrations/mismo.py
class MISMOExporter:
    async def export_ulad(self, application_id: UUID) -> str:
        """Generates ULAD (Uniform Loan Application Dataset) XML for DU/LPA submission.
        Maps application fields to MISMO v3.4 XPath locations."""
        ...

    async def export_uldd(self, application_id: UUID) -> str:
        """Generates ULDD (Uniform Loan Delivery Dataset) XML for loan delivery."""
        ...

    async def validate_mismo(self, xml_content: str, schema_version: str) -> list[str]:
        """Validates generated XML against MISMO XSD schemas."""
        ...
```

Field mapping: application/party/collateral JSONB fields map to MISMO XPath locations. The mapping table is maintained in a configuration file (`mismo_field_mapping.json`).

**Testing**:
- Unit: ULAD export produces well-formed XML with correct namespace
- Unit: Borrower data maps to correct MISMO PARTY container paths
- Unit: Property data maps to COLLATERAL/SUBJECT_PROPERTY
- Integration: Generated XML validates against MISMO v3.4 XSD
- Fixture: Expected XML output for a standard conventional mortgage application

#### 12.2 — Webhook System for Third-Party Integrations

**What**: Build a webhook delivery system that notifies external systems of loan lifecycle events.

**Design**:

```python
# src/services/webhook_service.py
class WebhookService:
    async def register_webhook(
        self, url: str, events: list[str], secret: str
    ) -> Webhook: ...

    async def deliver(
        self, event_type: str, payload: dict
    ) -> None:
        """Fan-out delivery to all registered webhooks for this event type.
        Uses HMAC-SHA256 signature for verification.
        Retry with exponential backoff on failure (3 attempts)."""
        ...
```

Webhook event types:
- `application.submitted`, `application.approved`, `application.denied`, `application.funded`
- `document.uploaded`, `document.verified`
- `decision.rendered`
- `condition.cleared`
- `disclosure.issued`

API endpoints:
- `POST /api/v1/webhooks` — Register webhook endpoint
- `GET /api/v1/webhooks` — List registered webhooks
- `DELETE /api/v1/webhooks/{id}` — Remove webhook
- `GET /api/v1/webhooks/{id}/deliveries` — Delivery history with status

**Testing**:
- Integration (mocked target): Webhook delivered with correct HMAC-SHA256 signature
- Integration: Failed delivery retried with exponential backoff
- Integration: Delivery history records success/failure status
- Unit: HMAC signature verification validates correctly

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Scaffolding          ─── required by everything
    │
Phase 2: Core Data Model & CRUD           ─── requires Phase 1
    │
Phase 3: Auth, RBAC & Security            ─── requires Phase 2
    │
Phase 4: Document Management              ─── requires Phase 3
    │
Phase 5: Workflow Engine & Tasks           ─── requires Phase 3
    │   (Phases 4 and 5 can be developed concurrently)
    │
Phase 6: Credit & Underwriting Engine      ─── requires Phase 3, Phase 4 (for document data)
    │
Phase 7: Compliance & Regulatory           ─── requires Phase 6 (for decisions), Phase 4 (for disclosures)
    │
Phase 8: AI Document Intelligence          ─── requires Phase 4 (document storage), Phase 6 (income for decisioning)
    │
Phase 9: Collateral & Fees                 ─── requires Phase 2 (application model), Phase 7 (TRID)
    │   (Phase 9 can be developed concurrently with Phase 8)
    │
Phase 10: Analytics Dashboard & Frontend   ─── requires Phases 2-7 (all API endpoints)
    │
Phase 11: Advanced AI Features             ─── requires Phases 6, 8, 10
    │   (11.1, 11.2, 11.3 can be developed concurrently)
    │
Phase 12: MISMO & Integrations            ─── requires Phase 2 (data model), Phase 7 (compliance)
    (Phase 12 can be developed concurrently with Phase 10 or 11)
```

Parallelism opportunities:
- Phases 4 and 5 can be developed concurrently after Phase 3
- Phases 8 and 9 can be developed concurrently after Phase 6
- Phase 11 subtasks (11.1, 11.2, 11.3) can be developed concurrently
- Phase 12 can be developed concurrently with Phase 10 or 11

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specifications.
2. All unit tests pass (`pytest tests/unit/ -v` exits 0).
3. All integration tests pass (`pytest tests/integration/ -v` exits 0).
4. Ruff linting passes (`ruff check src/ tests/` exits 0).
5. Ruff formatting passes (`ruff format --check src/ tests/` exits 0).
6. mypy type checking passes (`mypy src/` exits 0).
7. Docker build succeeds (`docker build -t los .` exits 0).
8. Alembic migrations apply cleanly on a fresh database (`alembic upgrade head` exits 0).
9. Feature works end-to-end when tested manually via the API (Swagger UI or curl).
10. New configuration options documented in `.env.example`.
11. New API endpoints appear in the auto-generated OpenAPI specification.
12. Audit logging active for all data-mutating operations.
13. RBAC enforced on all new endpoints.
14. No secrets, credentials, or PII in committed code or test fixtures.
