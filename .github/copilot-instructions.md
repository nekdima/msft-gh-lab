# Copilot Instructions for Inventory Management App

## 🏗 Project Architecture

This is a monorepo containing a full-stack application deployed to Azure Container Apps.

- **Frontend**: React 18 + Vite 5 + TypeScript (`/frontend`) — served on port 3000 in dev
- **Backend**: Python FastAPI with `uv` (`/backend`) — served on port 8000
- **Infrastructure**: Azure Bicep managed by `azd` (`/infra`, `azure.yaml`)
- **Database**: Azure Cosmos DB (NoSQL)
- **Data flow**: Browser → Frontend (`:3000/api/*`) → Vite proxy strips `/api` → Backend (`:8000/devices`)
- **Prod data flow**: Browser → Nginx → rewrites `/api/*` via `$BACKEND_URL` → Backend container

## 🔧 Backend Development (`/backend`)

- **Framework**: FastAPI with async routes. All route handlers are `async def`.
- **Dependency Management**: Uses `uv` and `pyproject.toml` (hatchling build, packages `["src"]`).
- **Schemas**: All DTOs live in `src.schemas` — `DeviceCreate`, `DeviceUpdate`, `DeviceResponse` (Pydantic v2, `from_attributes = True`).
- **Repository Pattern**:
  - **CRITICAL**: Never access the DB directly in routes. Routes call `src.repositories` module-level convenience functions (e.g., `list_devices()`, `create_device()`), which delegate to the active repository via `get_repository()`.
  - **Interface**: `src.repositories.base.DeviceRepository` — 5 abstract async methods.
  - **Dual Implementation**: You **MUST** implement both `CosmosDeviceRepository` (`cosmos_repository.py`) and `InMemoryDeviceRepository` (`memory_repository.py`) for any new data method.
  - **Factory**: `src.repositories.factory.get_repository()` — singleton, switches on `STORAGE_MODE` env var (`"cosmos"` default, `"memory"` for test/dev). Raises `ValueError` for unknown modes.
  - **Cosmos specifics**: Partition key = `device_id`, point reads use `read_item(item=id, partition_key=id)`, catches `CosmosResourceNotFoundError` → returns `None`/`False`.
  - **In-memory specifics**: Stores `DeviceResponse` objects in a `Dict[str, DeviceResponse]`, sorts by `created_at` descending on list.
- **Error Handling**: Raise `HTTPException` in routes. Repositories return `None` or `False` on not-found — never raise HTTP errors.
- **Running Locally**:
  ```bash
  cd backend
  uv sync
  STORAGE_MODE=memory uv run uvicorn src.main:app --reload
  ```

## ⚛️ Frontend Development (`/frontend`)

- **Structure**: React 18 / Vite 5 / TypeScript. No state library — uses `useState` hooks.
- **Components**: `App.tsx` (CRUD + state), `DeviceForm.tsx` (add/edit form), `DeviceList.tsx` (list display).
- **API Communication**:
  - Base URL: `import.meta.env.VITE_API_URL || '/api'` — all fetches go to `${API_URL}/devices`.
  - **Dev**: `vite.config.ts` proxies `/api` → `http://localhost:8000` with path rewrite removing `/api`.
  - **Prod**: Nginx (`nginx.conf.template`) uses `$BACKEND_URL` env var to proxy `/api/*`.
- **UI patterns**: Delete uses native `confirm()` dialog. Form input IDs: `name`, `assignedTo`.
- **Running Locally**:
  ```bash
  cd frontend
  npm install
  npm run dev
  ```

## ☁️ Infrastructure & Deployment (`/infra`)

- **Tooling**: Azure Developer CLI (`azd`). `azure.yaml` defines services `backend` (Python/Docker) and `frontend` (JS/Docker).
- **Hooks**: `preprovision` → `infra/hooks/preprovision.sh` runs before provisioning.
- **Deployment**: `azd up` provisions resources and deploys code.
- **Environment**: Backend uses System-Assigned Managed Identity for Cosmos DB (RBAC, no connection strings).
- **Naming convention**: `rg-gh-lab-${environmentName}`, `cosmos-${resourceToken}`, `cae-${resourceToken}`.
- **Automation**: Users should be able to set up everything by running `azd up` without additional scripts.

## 📝 Coding Conventions

- **Pydantic**: Use `src.schemas` for all DTOs. `DeviceResponse` has `id: str`, `created_at: datetime`, `updated_at: datetime`.
- **Async/Await**: The entire backend stack is async. All DB operations must be awaited.
- **IDs**: Generated with `str(uuid.uuid4())`. Cosmos DB uses string IDs.
- **Timestamps**: `datetime.now(timezone.utc).isoformat()` for Cosmos, `datetime.now(timezone.utc)` for in-memory.
- **Logging**: Use `logging.getLogger(__name__)` — standard Python logging throughout.
- **Imports**: Backend uses relative-style `src.` imports (e.g., `from src.schemas import ...`).

## 📍 Key Files

- **Service definitions**: [`azure.yaml`](azure.yaml)
- **Repo Interface**: [`backend/src/repositories/base.py`](backend/src/repositories/base.py)
- **Repo Factory**: [`backend/src/repositories/factory.py`](backend/src/repositories/factory.py)
- **Data Schemas**: [`backend/src/schemas.py`](backend/src/schemas.py)
- **API Routes**: [`backend/src/main.py`](backend/src/main.py)
- **Frontend Proxy**: [`frontend/vite.config.ts`](frontend/vite.config.ts)
- **Frontend Types**: [`frontend/src/types.ts`](frontend/src/types.ts)

## 🔑 Environment Variables

| Variable                   | Default      | Used By       | Purpose                             |
| -------------------------- | ------------ | ------------- | ----------------------------------- |
| `STORAGE_MODE`             | `cosmos`     | Backend       | `"cosmos"` or `"memory"`            |
| `COSMOS_ENDPOINT`          | _(required)_ | Backend       | Cosmos DB endpoint URL              |
| `COSMOS_DB_NAME`           | `inventory`  | Backend       | Cosmos database name                |
| `COSMOS_DEVICES_CONTAINER` | `devices`    | Backend       | Cosmos container name               |
| `ALLOWED_ORIGINS`          | `*`          | Backend       | CORS origins (comma-separated)      |
| `VITE_API_URL`             | `/api`       | Frontend      | API base URL in browser             |
| `BACKEND_URL`              | _(nginx)_    | Frontend prod | Backend URL for nginx reverse proxy |

## 🧪 Testing

- **Framework**: Playwright (sync API) with `pytest-playwright` for E2E tests.
- **Test location**: `tests/e2e/test_devices.py` — tests run against `http://localhost:3000`.
- **Dependencies**: `tests/requirements.txt` — `pytest`, `pytest-playwright`, `playwright`.
- **Fixtures**: `conftest.py` provides `page` (per-test), `app` (navigates to app), `clean_state` (deletes all devices via UI).
- **Locator strategy**: Role-based locators first — `get_by_role("textbox", name="Device Name *")`, `get_by_role("button", name="Add Device")`.
- **Dialog handling**: `page.on("dialog", lambda dialog: dialog.accept())` for delete confirmations.
- **Running tests** (requires both backend and frontend running):
  ```bash
  cd backend && STORAGE_MODE=memory uv run uvicorn src.main:app &
  cd frontend && npm run dev &
  cd tests && pip install -r requirements.txt && python -m pytest e2e/
  ```
- **Guidance**: When writing or updating tests, STRICTLY follow the instructions in `.github/instructions/playwright-python.instructions.md`.
