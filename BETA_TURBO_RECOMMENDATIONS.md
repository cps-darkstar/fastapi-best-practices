# FastAPI Best Practices — Recommendations for Beta-turbo

Targeted recommendations for `PSS-DARPA-turboFCL/Beta-turbo` based on its current profile and the patterns documented in this repository.

## Beta-turbo Profile

| Attribute | Status |
|-----------|--------|
| Framework | FastAPI |
| Structure | Domain-based (25+ modules) |
| Team | 8+ developers |
| DB (routes) | Async driver (asyncpg / async engine) |
| DB (workers) | Sync driver (Celery) |
| HTTP client | httpx (async) |
| CPU work | Inline in route handlers |
| Testing | Async tests + full CI |

---

## Priority 1: Offload CPU-Heavy Work from Routes

**Impact: Critical — this is the #1 performance bottleneck.**

CPU-intensive tasks (ML inference, data processing) running inline in route handlers will block the event loop (if `async def`) or exhaust the threadpool (if `def`). Either way, throughput degrades for all users under load.

### For heavy work (> 1 second): Use Celery

You already have Celery infrastructure. Move CPU-heavy operations to task workers and return a task ID.

```python
# BEFORE — blocks event loop or threadpool
@router.post("/analyze")
async def analyze_data(data: AnalysisRequest):
    result = heavy_ml_inference(data.payload)  # BLOCKS
    return {"result": result}

# AFTER — offload to Celery, return task ID
@router.post("/analyze", status_code=status.HTTP_202_ACCEPTED)
async def analyze_data(data: AnalysisRequest):
    task = heavy_ml_inference_task.delay(data.payload)
    return {"task_id": task.id}

@router.get("/analyze/{task_id}")
async def get_analysis_result(task_id: str):
    result = AsyncResult(task_id)
    if not result.ready():
        return {"status": "processing"}
    return {"status": "complete", "result": result.get()}
```

### For lightweight CPU work (< 1 second): Use ProcessPoolExecutor

When a full Celery round-trip is overkill, use a process pool to bypass the GIL:

```python
from asyncio import get_event_loop
from concurrent.futures import ProcessPoolExecutor

pool = ProcessPoolExecutor(max_workers=4)

@router.post("/quick-compute")
async def quick_compute(data: ComputeRequest):
    loop = get_event_loop()
    result = await loop.run_in_executor(pool, cpu_bound_fn, data.payload)
    return {"result": result}
```

> **Important:** Do NOT use `run_in_threadpool` for CPU work — Python's GIL makes threads useless for CPU-bound operations. Threads only help for I/O-bound blocking calls.

### Action Items
- [ ] Audit all routes for inline CPU work (ML model calls, heavy data transforms, transcoding)
- [ ] Move heavy operations (> 1s) to Celery tasks
- [ ] Use `ProcessPoolExecutor` for sub-second CPU work

---

## Priority 2: Dependency-Based Validation

**Impact: High — eliminates duplicated validation across 25+ domains.**

With 25+ domains, validation logic is likely duplicated across routes or missing entirely in some. FastAPI dependencies solve this when used as a validation layer, not just for injection.

### Shared dependency chains

```python
# src/shared/dependencies.py — reusable across all domains
async def valid_resource_by_id(resource_id: UUID4, service, not_found_exc):
    """Factory for resource existence checks."""
    resource = await service.get_by_id(resource_id)
    if not resource:
        raise not_found_exc()
    return resource

# src/projects/dependencies.py
async def valid_project_id(project_id: UUID4) -> dict:
    return await valid_resource_by_id(project_id, project_service, ProjectNotFound)

async def valid_owned_project(
    project: dict = Depends(valid_project_id),
    token_data: dict = Depends(parse_jwt_data),
) -> dict:
    if project["owner_id"] != token_data["user_id"]:
        raise NotProjectOwner()
    return project

# src/projects/router.py
@router.get("/projects/{project_id}")
async def get_project(project: dict = Depends(valid_project_id)):
    return project

@router.put("/projects/{project_id}")
async def update_project(
    update: ProjectUpdate,
    project: dict = Depends(valid_owned_project),  # chains existence + ownership
):
    return await service.update(project["id"], update)
```

### Key principles

- **Cached per request**: `valid_project_id` called 3 times in one request = 1 DB query
- **Chain small into complex**: existence → ownership → permissions
- **Consistent path variable names**: Use `{project_id}` everywhere, not `{proj_id}` in one place and `{project}` in another
- **Shared library**: With 25+ domains, maintain common validators in `src/shared/dependencies.py`

### Action Items
- [ ] Identify top 5-10 most duplicated validation patterns across domains
- [ ] Extract into shared chainable dependencies
- [ ] Enforce consistent path variable naming

---

## Priority 3: Code Organization at Scale

**Impact: Medium — reduces cognitive load across 8+ developers and 25+ domains.**

### Enforce explicit module-name imports

```python
# Good — unambiguous in a 25+ domain project
from src.auth import constants as auth_constants
from src.billing import service as billing_service

# Bad — which ERROR_CODE? Which service?
from src.auth.constants import ERROR_CODE
```

### Standardize domain file sets

Every domain should have the same files, even if some are thin. Reduces context-switching cost:

```
src/{domain}/
├── router.py        # API endpoints
├── schemas.py       # Pydantic models
├── models.py        # DB models
├── service.py       # Business logic
├── dependencies.py  # Route dependencies
├── config.py        # Domain-specific settings
├── constants.py     # Constants and error codes
└── exceptions.py    # Domain-specific exceptions
```

### Split BaseSettings per domain

A single config class with 25+ domains' worth of env vars is unmaintainable:

```python
# src/auth/config.py
class AuthConfig(BaseSettings):
    JWT_ALG: str
    JWT_SECRET: str
    JWT_EXP: int = 5

auth_settings = AuthConfig()

# src/billing/config.py
class BillingConfig(BaseSettings):
    STRIPE_KEY: str
    STRIPE_WEBHOOK_SECRET: str

billing_settings = BillingConfig()
```

### Custom base Pydantic model

Consistent serialization (datetime formats, shared methods) across all domains:

```python
from pydantic import BaseModel, ConfigDict

class CustomModel(BaseModel):
    model_config = ConfigDict(
        json_encoders={datetime: datetime_to_gmt_str},
        populate_by_name=True,
    )
```

### Action Items
- [ ] Audit cross-domain imports for consistency
- [ ] Check if BaseSettings is monolithic; split per domain if so
- [ ] Establish shared CustomModel base class if not already present

---

## Priority 4: SQL-First Data Processing

**Impact: Medium — reduces Python-side data manipulation and N+1 query patterns.**

### Push aggregation into SQL

```python
# Instead of fetching rows and assembling in Python:
select(
    posts.c.id,
    posts.c.title,
    func.json_build_object(
        text("'id', profiles.id"),
        text("'name', profiles.first_name"),
    ).label("creator"),
).select_from(
    posts.join(profiles, posts.c.owner_id == profiles.c.id)
)
```

### Migration hygiene (Alembic)

```ini
# alembic.ini — descriptive file names
file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(slug)s
```

```python
# Explicit naming conventions — don't rely on SQLAlchemy defaults
POSTGRES_INDEXES_NAMING_CONVENTION = {
    "ix": "%(column_0_label)s_idx",
    "uq": "%(table_name)s_%(column_0_name)s_key",
    "ck": "%(table_name)s_%(constraint_name)s_check",
    "fk": "%(table_name)s_%(column_0_name)s_fkey",
    "pk": "%(table_name)s_pkey",
}
```

### Database naming conventions

- `lower_case_snake` format, singular (`post`, not `posts`)
- Group by prefix: `payment_account`, `payment_bill`
- DateTime: `_at` suffix (`created_at`), Date: `_date` suffix (`birth_date`)

### Action Items
- [ ] Identify list/detail endpoints doing multi-query assembly in Python
- [ ] Refactor top offenders to use SQL aggregation
- [ ] Audit Alembic config for naming conventions and file template

---

## No Changes Needed

| Area | Current Status |
|------|---------------|
| Project structure | Domain-based ✓ |
| Async DB for routes | asyncpg / async engine ✓ |
| Sync DB for Celery | Correct pattern ✓ |
| HTTP client | httpx async ✓ |
| Testing | Async tests + full CI ✓ |

---

## Verification Checklist

Once changes are applied to Beta-turbo:

1. **Performance**: Load test routes that previously did inline CPU work — response times should improve significantly
2. **Validation**: Attempt invalid inputs at route boundaries — shared dependencies should catch them consistently
3. **Organization**: Run `ruff check` across the codebase; verify import consistency
4. **DB**: Compare query counts before/after SQL-first refactors (use SQLAlchemy echo or query logging)
5. **CI**: All existing async tests must continue to pass
