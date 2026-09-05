# AGENTS.md — OpenMetadata

Guidance for AI coding agents working in this repository. It assumes no prior knowledge of the
project. Deeper references: [CLAUDE.md](CLAUDE.md) (always-loaded session rules),
[ARCHITECTURE.md](ARCHITECTURE.md) (system map: modules, request/ingestion/search paths, invariants),
[DEVELOPER.md](DEVELOPER.md) (per-language workflows + end-to-end checklists),
[docs/index.md](docs/index.md) (knowledge index of design/plan/reference docs).

## Project overview

OpenMetadata is a unified metadata platform for data discovery, observability, and governance —
the "context layer for AI". It is a multi-module monorepo:

- **One Java backend** (`openmetadata-service`, Dropwizard/JAX-RS) fronting a **MySQL or
  PostgreSQL** catalog and an **Elasticsearch 7.17+ / OpenSearch 2.6+** index.
- A **React/TypeScript SPA** (`openmetadata-ui`) built with Vite.
- A **Python ingestion framework** (`ingestion/`, 120+ connectors) that feeds the catalog
  **through the same REST API** — ingestion is a client of the backend, not a separate path.
- **904 JSON Schemas** in `openmetadata-spec/` are the single source of truth; Java POJOs
  (jsonschema2pojo), Python Pydantic models (datamodel-code-generator), and TypeScript types
  (quicktype) are all **generated** from them. Never hand-edit generated output.
- Infrastructure: Apache Airflow orchestrates ingestion; everything ships via Docker.

Runtime request paths (see ARCHITECTURE.md for details):
- API request: `OpenMetadataApplication` (Jersey) → `service/resources/**` (JAX-RS) →
  `service/jdbi3/*Repository` → `jdbi3/CollectionDAO` → SQL; non-GET responses also fan out to
  change events (`service/events/`) and search indexing (`service/search/`).
- Ingestion: Python `<Name>Source` (loaded via `service_spec.py` / `ServiceSpec`) yields entities;
  the sink `ingestion/.../sink/metadata_rest.py` POSTs them to the REST API.
- Search: SPA `ui/src/rest/` → `resources/search/SearchResource` → `service/search/` → ES/OS via
  the shaded `es.*`/`os.*` clients.

## Technology stack

- **Backend**: Java 21 + Dropwizard, multi-module Maven (version `2.0.0-SNAPSHOT`), JDBI3 (not JPA).
- **Frontend**: React + TypeScript + **Vite** (dev server on :3000). Go-forward component library is
  `openmetadata-ui-core-components` (UntitledUI + Tailwind v4 with `tw:` prefix,
  react-aria-components); Ant Design + Less is the legacy stack (deprecated, migration ongoing).
- **Ingestion**: Python `>=3.10` (CI runs 3.10), Pydantic 2.x, setuptools (`pyproject.toml` +
  `setup.py`); package name `openmetadata-ingestion`.
- **DB/Search**: MySQL (default) or PostgreSQL; Elasticsearch 7.17+ or OpenSearch 2.6+ (shaded
  clients in `openmetadata-shaded-deps` — do not edit).
- **Orchestration**: Apache Airflow (`openmetadata-airflow-apis/`).
- **Package managers**: `mvn` (Java), `yarn` — never `npm` — (frontend), `pip` in a venv (Python).

## Repository layout

Maven modules:

| Module | Responsibility |
|---|---|
| `openmetadata-spec/` | 904 JSON Schemas + generated POJOs — the typing source of truth |
| `common/` | Shared utilities (`CommonUtil`, `nullOrEmpty`, …) |
| `openmetadata-shaded-deps/` | ES + OS clients relocated behind `es.*`/`os.*` (do not edit) |
| `openmetadata-service/` | Core backend: REST API, repositories, migrations runner, search, apps |
| `openmetadata-sdk/` | Java client SDK |
| `openmetadata-k8s-operator/` | Kubernetes operator |
| `openmetadata-mcp/` | MCP server exposing OpenMetadata to agents |
| `openmetadata-integration-tests/` | Backend API integration tests (`*IT.java`) |
| `openmetadata-ui-core-components/` | Canonical React component library |
| `openmetadata-ui/` | React SPA (app root: `src/main/resources/ui/`) |
| `openmetadata-dist/` | Packaging/assembly of the shippable server |
| `openmetadata-clients/` | Published client artifacts (isolated in the Maven graph) |

Other key trees:
- `ingestion/` — Python framework + connectors (`src/metadata/ingestion/source/{database,dashboard,
  pipeline,messaging,metadata,storage,search,mlmodel,api}/`), tests under `ingestion/tests/`.
- `openmetadata-airflow-apis/` — Python Airflow plugins (`openmetadata_managed_apis`).
- `bootstrap/sql/migrations/` — DB migrations (hybrid native + Flyway; see MIGRATION_SYSTEM.md).
- `conf/` — server configuration (`openmetadata.yaml`).
- `docker/` — local dev and production deployment compose files.
- `.claude/rules/*.md` — path-scoped rules that auto-load when you touch matching files (Java,
  frontend, ingestion, schemas, migrations, i18n, Playwright). Read the relevant rule before editing.
- `skills/` — procedural skills (planning, TDD, connector standards, checkstyles, etc.).
- Root config files: `pom.xml` (Maven aggregator), `Makefile` (dev tooling entry points),
  `package.json` (root, only carries quicktype for codegen), `ingestion/pyproject.toml`,
  `openmetadata-ui/src/main/resources/ui/package.json`, `.pre-commit-config.yaml`.

## Environment setup

- **Python venv is REQUIRED** before any Python work or `make generate`:
  ```bash
  python3.11 -m venv env && source env/bin/activate   # first time
  source env/bin/activate                              # every session
  ```
  In a git worktree the venv is NOT copied — create one or symlink the main repo's `env/`.
- **One-call setup (macOS + Linux)**: `make dev_setup` (idempotent). `make dev_check` diagnoses an
  existing checkout without changing it.
- **First-time bootstrap** (from the repo root — `make generate` is a root-only target):
  ```bash
  make prerequisites
  source env/bin/activate && cd ingestion && make install_dev_env && cd ..
  make generate                    # regenerate models after any schema change
  make yarn_install_cache
  make install_test precommit_install   # install the commit-time format/license gate
  ```
- **Docker dev services**: `docker compose -f docker/development/docker-compose.yml up -d`.

## Build and test commands

### Java backend
```bash
mvn clean install -DskipTests                      # build all modules
mvn clean install -pl openmetadata-spec            # regenerate Java POJOs after a schema change
mvn spotless:apply                                 # format — run before every commit
mvn test -pl openmetadata-service                  # module unit tests
mvn test -pl openmetadata-integration-tests        # backend API ITs (*IT.java)
```

### Frontend (`openmetadata-ui/src/main/resources/ui/`)
```bash
yarn yarn_install_cache      # (via root Makefile) install deps
yarn start                   # dev server on :3000
yarn lint:fix
yarn test                    # Jest unit tests
yarn playwright:run          # Playwright E2E (see playwright.config.ts)
yarn parse-schema            # after any connection schema change
yarn i18n                    # after touching translation keys
yarn license-header-fix      # Apache-2.0 headers on TS/TSX
```

### Python ingestion (`ingestion/`)
```bash
make install_dev             # install with dev dependencies (venv must be active)
make unit_ingestion          # pytest unit tests
make run_ometa_integration_tests
make py_format && make py_format_check   # ruff
make static-checks           # basedpyright (via nox, matches CI)
```

### Code generation (schema-first)
1. Edit the JSON schema in `openmetadata-spec/src/main/resources/json/schema/`.
2. `make generate` (Python models, root-only target).
3. `mvn clean install -pl openmetadata-spec` (Java POJOs).
4. `yarn parse-schema` (UI connection form schemas only).
5. Add a migration under `bootstrap/sql/migrations/native/{version}/` if the DB shape changed.

`make harness-check` warns (never blocks) on dead references in agent-facing docs.

## Code style guidelines

Hard constraints (apply to every language):

- **Schema-first.** JSON Schemas are the single source of truth. Edit the schema, then regenerate —
  never hand-edit generated code (`openmetadata-spec/target/`, `ingestion/src/metadata/generated/`,
  `openmetadata-ui/.../src/generated/`). The TypeScript output is committed and CI-enforced.
- **Migrations are append-only.** Never edit a shipped migration; add a new version with **both**
  MySQL and PostgreSQL variants; use the native path (`bootstrap/sql/migrations/native/`), make
  statements idempotent. See `.claude/rules/migrations.md`.
- **Bounded caches only.** No bare `dict`/`HashMap` as a cache without a size cap — they OOM on
  large catalogs. Python: `functools.lru_cache(maxsize=N)` / `cachetools.LRUCache`. Java:
  Caffeine/Guava `maximumSize(N)`. TypeScript: `lru-cache`.
- **Comments explain *why*, never restate code.** Only comment non-obvious logic, public-API docs,
  or `TODO`/`FIXME` with a ticket reference.
- **License headers are per-module — copy from a sibling file.** UI TS/TSX: Apache-2.0 (enforced by
  pre-commit + CI). `ingestion/` and `openmetadata-airflow-apis/` Python: Collate Community License
  1.0. Java: Apache-2.0, most files carry none.

Per-language rules:

- **Java**: Google style via spotless (120 col). Methods ≤ 15 lines, no magic strings, no wildcard
  imports, no convoluted if/else. Entities: resources extend `EntityResource`, repositories extend
  `EntityRepository`, DAOs hang off `CollectionDAO`; entity type constants come from `Entity.java`
  (never raw strings). See `.claude/rules/java.md`.
- **TypeScript/React**: prefer `openmetadata-ui-core-components` over Ant Design for new work; all
  Tailwind classes use the `tw:` prefix; colors via CSS custom properties/design tokens; no `any`;
  no string literals in JSX — use `t('label.key')` with kebab-case keys in
  `locale/languages/en-us.json`; import API types from `generated/`. New files: one stem + role
  suffix (`GlossaryList.tsx`, `.types.ts`, `.utils.ts`, `.test.tsx`); layers stay top-level
  (`components/`, `pages/`, `rest/`, `utils/`, `hooks/`). See
  `openmetadata-ui/src/main/resources/ui/DEVELOPER_HANDBOOK.md` and `.claude/rules/frontend-*.md`.
- **Python**: pytest style (plain `assert`, no `unittest.TestCase`); connectors follow the
  ServiceSpec/topology pattern (`{__init__, service_spec, metadata, connection}.py`,
  `<Name>Source` with `create()` raising `InvalidSourceException`); stream with generators, never
  accumulate; paginate REST calls; one `requests.Session()` per connector lifetime; keep
  connector-specific logic in the connector's own directory. See `.claude/rules/python-ingestion.md`.
- Extend the established design patterns (Template Method for repositories, Factory/Registry,
  Strategy/Adapter/Observer, ingestion Source→Sink pipeline) rather than inventing parallel ones —
  canonical examples in `docs/design-patterns.md`.

## Testing instructions

Philosophy: **test real behavior, not mock wiring.** Prefer integration tests over heavily-mocked
unit tests; mocks are for boundaries (HTTP clients, third-party APIs), not internals. Assert on
observable outcomes (API responses, DB state). No `Thread.sleep()` in tests — use condition-based
waiting.

- **Java**: every new REST endpoint requires an `*IT.java` in `openmetadata-integration-tests/`
  (run with real app, Docker, real OpenSearch). Coverage target: 90% of changed classes.
- **Python**: pytest under `ingestion/tests/` (`make unit_ingestion`); connector tests live in
  `tests/unit/topology/{type}/test_{connector}.py`.
- **UI**: Jest for components (`yarn test`); Playwright E2E for user-facing changes
  (`ui/playwright/`, config `playwright.config.ts`); zero-flakiness conventions in
  `.claude/rules/frontend-playwright.md`.
- **Pre-commit** (`.pre-commit-config.yaml`, install via `make install_test precommit_install`):
  on `git commit` runs Java format (spotless), Python format (ruff), UI format (prettier),
  design-token, and license-header checks on changed files, matching CI. Do not bypass with
  `--no-verify`.

## Security considerations

- **Never commit secrets** — use environment variables or a secrets manager. Auth is JWT with
  OAuth2/SAML; RBAC lives in the Java backend (`service/security/`); server config in
  `conf/openmetadata.yaml`.
- **Do not modify `.github/workflows/**` without explicit user authorization** — CI workflows are a
  supply-chain surface, guarded by a hook (`CLAUDE_ALLOW_WORKFLOW_EDITS=1` to authorize).
- All mutating REST methods must call `authorizer.authorize(...)` before execution; change events
  mask PII before persistence.
- `conf/private_key.der` / `conf/public_key.der` exist for local dev JWT signing — treat as
  sensitive material; never reuse for real deployments.

## Deployment

- **Local dev stack**: `docker compose -f docker/development/docker-compose.yml up -d` (MySQL,
  Elasticsearch, Airflow, server).
- **Quickstart/full product**: `docker/docker-compose-quickstart/`,
  `docker/docker-compose-openmetadata/`.
- **Production packaging**: `openmetadata-dist/` assembles the server; run scripts in `bin/`
  (`openmetadata.sh`, `openmetadata-server-start.sh`); ops tooling in `bootstrap/openmetadata-ops.sh`.
- **Kubernetes**: helm/operator artifacts under `docker/` and `openmetadata-k8s-operator/`.

## Adding things (end-to-end checklists in DEVELOPER.md)

- **New entity**: schema + API schema → generate → `Entity` constant → `CollectionDAO` method →
  `EntityRepository` → Mapper → `EntityResource` → migration → search index → `*IT.java` → UI
  types/API/components → i18n keys.
- **New connector**: connection schema → service-type enum → generate → ClassConverter (if `oneOf`)
  → Python source package (`connection.py`, `metadata.py`, ServiceSpec) → unit tests →
  `yarn parse-schema` + UI wiring.
