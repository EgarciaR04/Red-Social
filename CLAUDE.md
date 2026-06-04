# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stack

- **Backend**: Python / Flask, MongoDB (PyMongo), JWT (flask-jwt-extended), Pydantic validation, Flasgger (Swagger docs)
- **Frontend**: Angular 21 (module-based), Angular Material
- **Database**: MongoDB running in Docker

## Running the project

**Database** (start first):
```bash
docker compose up -d
```
Mongo Express UI is available at `http://localhost:8081`.

**Backend** (from project root, with the venv activated):
```bash
cd backend && source venv/bin/activate && cd ..
python run.py
```
Runs on `http://localhost:5000`. Swagger docs at `http://localhost:5000/docs/`.

**Frontend** (from `frontend/`):
```bash
cd frontend && npm start
```
Runs on `http://localhost:4200`.

**Frontend tests**:
```bash
cd frontend && npm test
```

## Backend architecture

The backend uses the **Application Factory** pattern (`backend/app/__init__.py:create_app`). Extensions (`mongo`, `jwt`, `swagger`, `cors`) are instantiated without an app in `extensions.py` and bound later via `init_app()` to avoid circular imports.

Each API domain follows a 3-layer pattern:
```
routes.py → service.py → repository.py
```
- **routes**: Flask blueprint, HTTP in/out only, delegates to service
- **service**: Business logic, Pydantic validation, password hashing
- **repository**: Direct PyMongo calls, no business logic

Blueprints registered in `_register_blueprints`:
- `/auth` → register, login (public)
- `/api/users` → user data (JWT-protected via `@jwt_required()`)

Pydantic schemas (`schemas.py`) validate request data before any DB access. Validation errors are returned as `{"ok": false, "errors": [...]}`.

Config is selected by `FLASK_ENV` env var (`development` / `production` / `testing`). All env vars are loaded from `.env` at the root.

## Frontend architecture

Angular **module-based** (not standalone components). Lazy loading is used: the root router loads `AuthModule` at the `/auth` path.

Key conventions:
- `SharedImportModule` (`app/shared-import/`) aggregates all Angular Material imports — add new Material modules there and import `SharedImportModule` in feature modules.
- `Config` service (`service/config/config.ts`) builds the API base URL from `environment.ts`. All HTTP services should use `configService.appConfig.apiUrl` as the base.
- Backend URL is set in `frontend/src/environment/environment.ts` (`server: 'http://localhost:5000'`).

## Adding a new API endpoint

1. Create `backend/app/api/<domain>/` with `__init__.py`, `routes.py`, `service.py`, `repository.py`, `schemas.py`
2. Register the blueprint in `backend/app/__init__.py:_register_blueprints`
3. Add Flasgger YAML doc in `docs/<endpoint>.yml` and decorate the route with `@swag_from`

## Adding a new frontend feature

1. Create a feature module under `app/components/<feature>/`
2. Add a lazy-loaded route in `app-routing-module.ts`
3. Import `SharedImportModule` for Angular Material components
