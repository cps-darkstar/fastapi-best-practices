# FastAPI Best Practices — Recommendations for Beta-turbo

Targeted recommendations for `PSS-DARPA-turboFCL/Beta-turbo` based on patterns documented in this repository, grounded against the actual repo tree via peer review.

**Lane**: Behavioral (guidance, not truth mapping)
**Suggested home in Beta-turbo**: `.cursor/rules/fastapi-performance-patterns.md` or extension of `.cursor/rules/fastapi-async-patterns.md`

## Beta-turbo Profile

| Attribute | Actual |
|-----------|--------|
| Framework | FastAPI |
| Structure | Domain-based — **58 modules** under `apps/backend/modules/`, ~160 service files under `apps/backend/services/` |
| Team | 8+ developers (verify against current contributors) |
| DB (routes) | Async driver (asyncpg / async engine) |
| DB (workers) | Sync driver (Celery — `apps/backend/worker_tasks/`) |
| HTTP client | httpx (async) |
| CPU work | Inline in route handlers — AI/inference paths in `apps/backend/services/ai*`, `apps/backend/services/agents/`, `apps/backend/services/foci/` |
| Testing | Async tests + full CI |
| Migrations | 208 versions in `database/migrations/backend/`, two alembic.ini files |
| Config | `apps/backend/config/` (3 files, 20 KB), `apps/backend/core/rainbow_config.py` (main config source) |
| Validation | `apps/backend/modules/validation/` — 26 files, 262 KB (largest module) |
| TOM | `docs/architecture/TOM.md` (PR #1104) — 178 unmapped stray files at R00 |

---

## Priority 1: Offload CPU-Heavy Work from Routes

**TOM rows**: R08 (TRANSFORM: AI) + R03 (TRANSFORM: API Surfaces)

**Impact: Critical — this is the #1 performance bottleneck.**

CPU-intensive tasks (ML inference, data processing) running inline in route handlers will block the event loop (if `async def`) or exhaust the threadpool (if `def`). Either way, throughput degrades for all users under load. For a DARPA-contract platform serving government users, this is a reliability concern, not just a performance one.

### Where to audit

Check these locations for direct inline calls to AI/inference services:
- **Route handlers**: `apps/backend/api/v1/` (87 items) and `apps/backend/modules/*/api.py` (58 modules)
- **AI services called**: `apps/backend/services/ai*`, `apps/backend/services/agents/`, `apps/backend/services/foci/`
- **Existing Celery infra**: `apps/backend/worker_tasks/` — offload target already exists

Look for patterns like:
```python
# In any api.py or route file — direct calls to AI services
from apps.backend.services.ai_something import run_inference

@router.post("/endpoint")
async def some_route(data: Request):
    result = run_inference(data)  # If this is CPU-bound, it blocks
    return result
```

### For heavy work (> 1 second): Use Celery

Celery infrastructure already exists at `apps/backend/worker_tasks/`. Move CPU-heavy operations there and return a task ID.

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
- [ ] Grep `apps/backend/api/v1/` and `apps/backend/modules/*/api.py` for imports from `services/ai*`, `services/agents/`, `services/foci/`
- [ ] For each match, determine if the call is CPU-bound or I/O-bound (Bedrock API call = I/O, local inference = CPU)
- [ ] Move CPU-bound operations to `apps/backend/worker_tasks/`
- [ ] Use `ProcessPoolExecutor` for sub-second CPU work that doesn't justify Celery overhead

---

## Priority 2: Dependency-Based Validation

**TOM rows**: R11 (VERIFY: Validation) + R03 (TRANSFORM: API Surfaces)

**Impact: High — the validation module (`apps/backend/modules/validation/`, 26 files, 262 KB) is the largest in the repo. Shared FastAPI dependencies could thin it significantly.**

With 58 modules each having their own `api.py`, validation logic is likely duplicated or inconsistent. FastAPI's `Depends()` system with per-request caching can consolidate this.

### Where to audit

- **Current validation**: `apps/backend/modules/validation/` (26 files, 262 KB) — understand what's in here first
- **Route files**: `apps/backend/modules/*/api.py` — check for inline validation that should be dependencies
- **Cross-module patterns**: Look for repeated existence checks, permission checks, ownership verification

### Shared dependency chains

```python
# apps/backend/shared/dependencies.py — reusable across all 58 modules
async def valid_resource_by_id(resource_id: UUID4, service, not_found_exc):
    """Factory for resource existence checks."""
    resource = await service.get_by_id(resource_id)
    if not resource:
        raise not_found_exc()
    return resource

# apps/backend/modules/projects/dependencies.py
async def valid_project_id(project_id: UUID4) -> dict:
    return await valid_resource_by_id(project_id, project_service, ProjectNotFound)

async def valid_owned_project(
    project: dict = Depends(valid_project_id),
    token_data: dict = Depends(parse_jwt_data),
) -> dict:
    if project["owner_id"] != token_data["user_id"]:
        raise NotProjectOwner()
    return project

# apps/backend/modules/projects/api.py
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
- **Shared library**: With 58 modules, maintain common validators in a shared location

### Action Items
- [ ] Audit `apps/backend/modules/validation/` to understand current patterns and what can be replaced
- [ ] Identify top 5-10 most duplicated validation patterns across the 58 modules' `api.py` files
- [ ] Extract into shared chainable dependencies
- [ ] Enforce consistent path variable naming across all 58 modules

---

## Priority 3: Code Organization at Scale

**TOM rows**: R06 (TRANSFORM: Services) + R15 (TRANSFORM: Backend Core)

**Impact: Medium-High — at 58 modules and ~160 service files, this is significantly larger than originally estimated. Organization patterns matter more, not less, at this scale.**

### Enforce explicit module-name imports

Critical with 58 modules — ambiguous imports waste developer time:

```python
# Good — unambiguous across 58 modules
from apps.backend.modules.auth import constants as auth_constants
from apps.backend.modules.billing import service as billing_service

# Bad — which ERROR_CODE? Which service?
from apps.backend.modules.auth.constants import ERROR_CODE
```

### Check BaseSettings situation

- Current config: `apps/backend/config/` (3 files, 20 KB) + `apps/backend/core/rainbow_config.py` (main source)
- If `rainbow_config.py` is monolithic, consider splitting domain-specific settings to their respective modules
- Minor impact relative to P1/P2, but worth a quick check

```python
# Per-domain config pattern
# apps/backend/modules/auth/config.py
class AuthConfig(BaseSettings):
    JWT_ALG: str
    JWT_SECRET: str

auth_settings = AuthConfig()
```

### Custom base Pydantic model

Consistent serialization (datetime formats, shared methods) across all 58 modules:

```python
from pydantic import BaseModel, ConfigDict

class CustomModel(BaseModel):
    model_config = ConfigDict(
        json_encoders={datetime: datetime_to_gmt_str},
        populate_by_name=True,
    )
```

### Action Items
- [ ] Audit cross-module imports for consistency
- [ ] Check if `apps/backend/core/rainbow_config.py` is monolithic; assess split cost/benefit
- [ ] Establish shared CustomModel base class if not already present
- [ ] Address TOM R06 service split disposition for `apps/backend/services/` (~160 files)

---

## Priority 4: SQL-First Data Processing

**TOM row**: R14 (DEFINE: Database)

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

With 208 migration versions in `database/migrations/backend/` and two alembic.ini files (`database/alembic.ini` primary, `apps/backend/alembic.ini` local dev), consistency is critical.

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

### Action Items
- [ ] Check naming convention consistency across both alembic.ini files
- [ ] Audit the 208 migration versions for naming pattern adherence
- [ ] Identify list/detail endpoints doing multi-query assembly in Python
- [ ] Refactor top offenders to use SQL aggregation

---

## Known Structural Debt (Do NOT Mark as "No Changes Needed")

The foundational architectural choices (async DB, domain structure, httpx, Celery) are correct. However, the implementation has significant drift that should not be papered over:

| Area | Status | Notes |
|------|--------|-------|
| Project structure | Domain-based ✓ | But TOM R00 shows **178 unmapped stray files** |
| Async DB for routes | Correct pattern ✓ | Needs audit for blocking calls in async routes |
| Sync DB for Celery | Correct pattern ✓ | — |
| HTTP client | httpx async ✓ | — |
| Testing | Async tests + full CI ✓ | — |
| Services | ~160 files | TOM R06: needs split disposition |
| OSCAL | 32.5% of repo | TOM R07: significant structural weight |

These are out of scope for this behavioral-lane document but should not be ignored.

---

## TOM Cross-Reference

| Priority | Recommendation | TOM Row(s) |
|----------|---------------|-------------|
| P1 | CPU offload from routes | R08 (TRANSFORM: AI) + R03 (TRANSFORM: API Surfaces) |
| P2 | Dependency-based validation | R11 (VERIFY: Validation) + R03 |
| P3 | Code organization at scale | R06 (TRANSFORM: Services) + R15 (TRANSFORM: Backend Core) |
| P4 | SQL-first data processing | R14 (DEFINE: Database) |

---

## Verification Checklist

Once changes are applied to Beta-turbo:

1. **Performance**: Load test routes that previously did inline CPU work — response times should improve significantly
2. **Validation**: Attempt invalid inputs at route boundaries — shared dependencies should catch them consistently
3. **Organization**: Run `ruff check` across the codebase; verify import consistency
4. **DB**: Compare query counts before/after SQL-first refactors (use SQLAlchemy echo or query logging)
5. **CI**: All existing async tests must continue to pass
