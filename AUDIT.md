# OpenAPI Annotation Audit

Baseline audit of Backend OpenAPI annotations, done on 2026-04-20. Tells you what needs to change in `Backend/app/routers/` and `Backend/app/schemas/` to turn auto-generated Mintlify API reference into something Stripe-quality.

## Headline numbers

| Metric | Current | Target for Stripe-level |
|--------|---------|-------------------------|
| Routes with `summary` | 20% | ≥ 90% |
| Routes with narrative `description` | 20% | ≥ 90% |
| Routes with documented `responses` (error codes) | 0% | ≥ 90% |
| Routes with request/response examples | 3% | ≥ 70% |
| Pydantic fields with `Field(..., description=...)` | ~10% | ≥ 80% |

**Estimated effort to close the gap:** 2–3 full days for the top 10 files listed below. The rest is long-tail polish.

## Per-file scorecard

Audited 61 routes across 8 files. Grade = overall annotation coverage.

| File | Routes | Has summary | `response_model` | Param descriptions | Error docs | Grade |
|------|--------|-------------|------------------|---------------------|------------|-------|
| `routers/invites/user_invites.py` | 4 | 3/4 (75%) | 4/4 (100%) | 0/4 | 0/4 | **C+** |
| `routers/billing/billing_router.py` | 18 | 1/18 (6%) | 9/18 (50%) | 8/18 (44%) | 0/18 | **C-** |
| `routers/files/parse_router.py` | 4 | 1/4 (25%) | 3/4 (75%) | 0/4 | 0/4 | **D** |
| `routers/chat/session_router.py` | 9 | 1/9 (11%) | 4/9 (44%) | 0/9 | 0/9 | **D+** |
| `routers/safety/safety_router.py` | 5 | 1/5 (20%) | 1/5 (20%) | 0/5 | 0/5 | **D** |
| `routers/devices/devices.py` | 8 | 0/8 (0%) | 5/8 (63%) | 0/8 | 0/8 | **D** |
| `routers/agent/device_control.py` | 8 | 0/8 (0%) | 4/8 (50%) | 0/8 | 0/8 | **D** |
| `routers/message/message_router.py` | 5 | 0/5 (0%) | 2/5 (40%) | 0/5 | 0/5 | **D** |

### Schemas

| File | Pattern | Grade |
|------|---------|-------|
| `schemas/chat_message.py` | 0 examples, 0 field descriptions, bare `Dict/Optional` | **F** |
| `schemas/billing.py` | `BillingStatusResponse` is a 45-field mega-model with zero descriptions | **F** |
| `schemas/devices.py` | Only `E2BSandboxCreate` fully annotated. `DeviceRead`/`DeviceCreate` minimal. | **D** |
| `schemas/chat_session.py` | 1 example, 0 descriptions on 8 key fields | **D** |

## Best-in-class patterns — copy these

Three spots in the codebase already do it right. Use them as your template.

**1. `routers/invites/user_invites.py:15-32`** — clear 1-sentence docstrings on every endpoint.

**2. `routers/chat/session_router.py:141-149`** — pagination parameters with `Query`.

**3. `schemas/chat_session.py:8`** — `Field` with an `example`:
```python
title: str = Field(..., min_length=1, max_length=200, example="Support conversation with bot")
```

## Worst offenders — fix these first

### 1. `routers/devices/devices.py` — public-facing, 0% documentation

8 endpoints. All lack `summary`, `description`, `responses`. `DeviceRead`/`DeviceCreate` fields have no descriptions. This is the API every desktop and mobile client hits — it must be first.

**Specifically:**
- Line 42 (`@router.post("")`): no summary, no description, no error responses
- Line 107 (`@router.get("")`): bare `Query` parameters, no descriptions
- Document `409` (KID conflict), `403` (forbidden), `404` (not found)

### 2. `routers/message/message_router.py` — chat surface, 0% documentation

5 endpoints, bare `Dict`/`Optional` fields. Users have no way to know what `meta_data` or `task_id` are for.

**Specifically:**
- Line 109 (`@router.post("")`): no summary
- Line 15–16 (`schemas/chat_message.py`): `content: Optional[str] = ""` and `meta_data: Optional[Dict] = {}` have no explanation
- Document `422` (billing/version check), `429` (rate limit), `503` (temp unavailable)

### 3. `routers/billing/billing_router.py` + `schemas/billing.py` — monetization gateway

18 endpoints crammed into one file, sparse docs. `BillingStatusResponse` is a 45-field mega-model that mixes plan info, freemium fields, model profiles, and Cloud PC — Mintlify will render an unusable 2-page schema.

**Specifically:**
- Split `BillingStatusResponse` into 3 models: `BillingStatusCore`, `PlanInfo`, `FeatureFlags`
- Add `description` to every field
- Add `json_schema_extra` examples per tier (free, standard)
- Document Stripe webhook flows at line 276+

### 4. `routers/agent/device_control.py` — agent contract invisible

8 endpoints, 0 docstrings. Action lifecycle (pending → completed/failed, Redis TTLs) is not visible to anyone reading the docs.

**Specifically:**
- Document `423` (device locked), `504` (action timeout), `503` (sandbox down)
- Mark internal-only endpoints with `tags=["internal"]` or `include_in_schema=False`

### 5. `routers/safety/safety_router.py` — safety-critical

5 endpoints. `/request-confirmation` flow (timeout, response format) needs explicit documentation or users will not know how to wire it.

## Recommended fix order

Priority is a mix of impact (how many users hit it) and leverage (how much quality it unlocks).

| # | File | Why |
|---|------|-----|
| 1 | `routers/devices/devices.py` | Every client hits this first |
| 2 | `schemas/billing.py` | Blocker for all billing endpoints |
| 3 | `routers/billing/billing_router.py` | Depends on #2 |
| 4 | `routers/message/message_router.py` | Chat UX surface |
| 5 | `schemas/chat_message.py` | Depends on #4 |
| 6 | `routers/agent/device_control.py` | Agent integration contract |
| 7 | `routers/safety/safety_router.py` | Safety-critical |
| 8 | `routers/files/parse_router.py` | Document `ALLOWED_TYPES`, size limits |
| 9 | `routers/chat/session_router.py` | Already best-in-class — add 2–3 examples |
| 10 | `routers/invites/user_invites.py` | Light touch — add `404` responses |

## Before and after

### Router (from `devices.py:42`)

**Before:**
```python
@router.post("", response_model=DeviceRead, status_code=201)
async def register_device(
    payload: DeviceCreate,
    response: Response,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_active_user),
):
    # No docstring, no summary, no error responses
    ...
```

**After (Stripe-style):**
```python
@router.post(
    "",
    response_model=DeviceRead,
    status_code=201,
    summary="Register or update a device",
    description=(
        "Register a new device or update an existing one. Automatically "
        "deduplicates by `machine_id` (hardware UUID) or `device_token_kid`. "
        "Soft-deleted devices are restored on re-registration."
    ),
    responses={
        201: {"description": "Device registered"},
        200: {"description": "Device already exists; restored if deleted"},
        409: {"description": "`device_token_kid` already registered to another user"},
        401: {"description": "Unauthorized"},
    },
    tags=["Devices"],
)
async def register_device(...):
    ...
```

### Schema (from `chat_message.py:13-16`)

**Before:**
```python
class ChatMessageCreate(BaseModel):
    role: MessageRole
    content: Optional[str] = ""
    meta_data: Optional[Dict] = {}
    task_id: Optional[str] = None
```

**After:**
```python
class ChatMessageCreate(BaseModel):
    role: MessageRole = Field(
        ...,
        description="Message sender: `user`, `assistant`, or `system`.",
    )
    content: Optional[str] = Field(
        "",
        description=(
            "Message text. Can be empty for multi-modal messages that carry "
            "only attachments."
        ),
    )
    meta_data: Optional[Dict] = Field(
        default={},
        description=(
            "User-visible metadata. Internal keys such as `worker_id` are "
            "filtered from responses."
        ),
    )
    task_id: Optional[str] = Field(
        None,
        description=(
            "Link to a task ID for tracking agent actions. Null for user messages."
        ),
    )

    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "role": "user",
                    "content": "Write a test for this function",
                    "meta_data": {"source": "web_ui"},
                    "task_id": None,
                },
                {
                    "role": "assistant",
                    "content": "Here's a test...",
                    "meta_data": {"agent_reasoning": "..."},
                    "task_id": "task_123",
                },
            ]
        }
    }
```

## Anti-patterns to avoid

- **Catch-all docstrings** — "Handles the thing." Tells the reader nothing.
- **Restating the function name** — `summary="Register device"` on `register_device()` adds zero information. Write outcome-level summaries: `"Register or update a device"`.
- **Undocumented `response_model=None`** — if you return `dict`, Mintlify will show `object` with no fields. Always define a response model.
- **Mega-models** — any `BaseModel` with > 20 fields should be split.
- **Shared `meta_data` dictionaries** — if every endpoint has a generic `Optional[Dict]`, document the expected keys via `Literal` unions or split into typed sub-models.

## Workflow for fixing a file

1. Pick a router file from the priority list.
2. For every `@router.*` decorator: add `summary`, `description`, `responses`, `tags`.
3. For every Pydantic schema it uses: add `Field(description=...)` per field, and `model_config` with examples.
4. Re-run the export script (`poetry run python scripts/export_openapi.py`) and `mint dev` in `mintlify-docs/` to preview.
5. Compare to the Stripe reference: is every field explained? Is every error code listed? Are examples realistic?

## Measuring progress

After each pass, count:

```bash
# Routes with summary
rg '@router\.[a-z]+' Backend/app/routers/ --count-matches
rg 'summary=' Backend/app/routers/ --count-matches

# Field descriptions
rg 'Field\(' Backend/app/schemas/ --count-matches
rg 'Field\([^)]*description=' Backend/app/schemas/ --count-matches
```

Track the ratio over time. Commit the numbers to this file on a monthly basis.
