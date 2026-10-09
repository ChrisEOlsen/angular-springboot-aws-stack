# Project instructions — Angular + Spring Boot + AWS

## Scope and decision rules

You are the build agent. Follow the stack and conventions below.

## Template decision status

Angular + Angular CLI + Tailwind CSS, Spring Boot, a Docker development container with browser-based code-server, Vim motions, and OpenAI Codex CLI, plus the existing AWS topology are requested. The Java, persistence, authentication, validation, and job-processing choices below are proposed defaults pending explicit user approval. Do not treat those proposals as approved or implement an application until they are resolved in Phase 1. Pin and document compatible supported versions in the approved plan; do not use floating dependency or image tags.

- For the initial application build, complete Phase 1 and Phase 2 before writing implementation code. Missing deployment credentials block initial implementation as well as launch.
- Read-only investigation, reviews, planning, and documentation edits do not require the build phases. Preparing placeholder configuration and the approved bootstrap prerequisites is also allowed before implementation.
- For subsequent features and fixes, reuse approved decisions. Ask about and obtain approval for material changes to scope, architecture, data handling, or design; do not restart discovery for routine maintenance.
- Before every launch, recheck Phase 2 for the target environment.
- An approved build plan does not authorize deployment or teardown. Deploy when the user requests it. Teardown requires the separate confirmation in Phase 5.
- If requirements conflict or a material decision is missing, explain the specific conflict and ask a focused question. Do not silently substitute technologies or relax security rules.
- Never commit secrets or print secret values. Inspect secret names, configuration keys, and presence only.

## Phase 1 — Discover and approve the plan

Ask focused questions, grouped by topic. Reuse answers already given. Resolve decisions that affect scope, architecture, security, or acceptance criteria; propose defaults for minor implementation details instead of asking indefinitely.

Cover:

- Purpose, users, roles, and core user flows. The application is single tenant.
- Features, stored data, relationships, retention, and deletion behavior.
- File types, sizes, access rules, and upload/download flows.
- Background work, recurring work, time zones, retry behavior, and failure handling.
- Pages, navigation, and visual design.
- Deploy environments, AWS account and region, domain and HTTPS, and launch acceptance criteria.

For design, suggest that the user ask their Muse agent to explore directions using the Figma connector and provide the resulting design or Figma link. Implement the supplied design. If none is supplied, ask for one or for explicit authorization to propose a design; do not invent a full visual design unprompted.

Write a reviewable plan covering:

1. Features and acceptance criteria.
2. Data model, indexes, and authorization rules.
3. REST API endpoints, validation, pagination, and errors.
4. Pages and the approved design reference.
5. Quartz jobs and recurring triggers, including retries and idempotency.
6. Infrastructure changes beyond the base topology, if any.
7. Development-container tooling, bind mounts, editor integration, ports, Compose modes, local file behavior, deployment transport to EC2, migration execution, HTTPS, and rollout/recovery procedure.
8. Required configuration and secret names for local development and each environment.
9. Bootstrap ownership and the launch checklist.

Obtain the user's explicit approval of the plan before implementation. Record the approved decisions in repository documentation without secrets.

## Phase 2 — Verify launch prerequisites

Secrets live in a gitignored `.env` for local development and GitHub Environment secrets for deployment. AWS deployment authentication uses GitHub Actions OIDC; never store long-lived AWS keys.

Verify before initial implementation and again before launch:

- `.env.example` is committed and contains placeholders only. `.env` exists locally, is gitignored, and has the required local values. Do not display its contents.
- Every approved deploy target has a GitHub Environment and the required secret names. Check with `gh secret list --env <name>`; listing names does not verify secret values.
- The GitHub OIDC provider `token.actions.githubusercontent.com` exists in the expected AWS account.
- Each environment has its own IAM role with the trust policy and permissions below.
- Local AWS access works through SSO / IAM Identity Center or a named profile using temporary credentials. Use `aws sts get-caller-identity` to verify the expected account. Never commit AWS credentials.
- The Terraform state bucket exists and the approved backend configuration identifies the correct bucket, region, and environment-specific state key.

If environments, the OIDC provider, roles, or required configuration are missing, report exactly what is missing and stop implementation and launch. Ask the user to create missing environments/provider/roles; do not create them implicitly. If access prevents verification, report the check as unverified rather than treating it as passed.

### OIDC trust and permissions

Each role must allow `sts:AssumeRoleWithWebIdentity` for the GitHub OIDC provider with `StringEquals` conditions:

- `token.actions.githubusercontent.com:aud` = `sts.amazonaws.com`
- `token.actions.githubusercontent.com:sub` = `repo:OWNER/REPO:environment:<env>`

Never wildcard the `sub` claim. Actions deployment jobs must declare their GitHub Environment, use `permissions: { id-token: write, contents: read }`, and assume that environment's role, for example with `aws-actions/configure-aws-credentials` and `role-to-assume`. Verify that CI assumes the expected account and role.

When roles are missing or insufficient, direct the user to AWS console → IAM → Identity providers to add GitHub as an OIDC provider, then Roles → Create role → Web identity. Explain the exact repo/environment trust scope above and show this permission list verbatim for the customer-managed policy:

- EC2: full (instances, security groups, key pairs, Elastic IPs). The Terraform operates in the account's default VPC — do not build custom networking unless a plan explicitly calls for it.
- RDS: full (Postgres instances, subnet groups, parameter groups).
- S3: full (app file buckets, plus the Terraform state bucket).
- ECR: full (image repositories).

If the approved deployment transport or other planned resources require additional permissions, identify them in the plan and ask the user to provision them. Do not assume the list grants permissions to other services.

### Bootstrap exception

Before implementation, the agent may prepare `.gitignore`, a placeholder-only `.env.example`, and bootstrap documentation. Local secret values must be supplied or generated without printing them.

The state bucket must exist before the application's first `terraform init`. The approved plan must name who creates it and how. The user may create it, or explicitly authorize a one-time bootstrap with local SSO. Keep its lifecycle separate from the application Terraform root; document the bootstrap and cleanup procedure. Application Terraform uses S3 remote state from its first initialization.

Values available only after provisioning, such as an RDS endpoint, must be identified as deployment outputs in the plan. Their absence is not a missing pre-provisioning secret. Required user-supplied secrets must exist before provisioning/deployment; outputs must be resolved before starting the application.

## Documentation lookup before implementation

Before writing code that uses a library or infrastructure API, verify the relevant API against official documentation for the selected version. This applies to Java, Maven, Spring Boot, Spring MVC, Spring Data JPA, Hibernate, Jakarta Bean Validation, Spring Security, Spring Session JDBC, Flyway, Quartz, AWS SDK for Java v2, Angular, Angular CLI, Tailwind, Terraform and its AWS provider, AWS services, Docker Compose, the Dev Container specification, code-server, and VSCodeVim.

1. Check for Context7 tools that resolve a library ID and retrieve documentation. Tool names can vary by integration.
2. Resolve the library, then retrieve documentation for the relevant topic and version before implementing it. Reuse verified documentation within the task unless the topic or version changes.
3. If Context7 is unavailable, tell the user and use official documentation as the fallback. For Claude Code, the installation command is `claude mcp add --transport http context7 https://mcp.context7.com/mcp`; other clients require their own MCP setup.
4. Flag anything that could not be verified. Do not invent APIs. Documentation determines API usage, but does not authorize changing the locked stack or approved plan.

## Phase 3 — Build

### Stack (backend defaults pending approval)

- Java 21, Maven with the Maven Wrapper, Spring Boot, and Spring MVC. Use an executable JAR with embedded Tomcat, not a separate application server. Pin a supported Spring Boot version compatible with Java 21 and use its dependency management.
- Spring Data JPA with Hibernate for persistence, Flyway for versioned SQL migrations, and Jakarta Bean Validation for request validation. No Drizzle, Zod, or Node backend.
- Spring Security for authentication and authorization, with Spring Session JDBC storing server-side sessions in Postgres. No Better Auth or separate Node authentication service.
- Quartz with a JDBC job store in Postgres for durable background and recurring work. No BullMQ or Redis by default. Quartz is the execution mechanism; durable business status and retry policy remain explicit application data and logic.
- Postgres 16: Docker Compose locally, RDS in deployed environments. Postgres is the system of record and the only datastore, including session and scheduling tables.
- S3 for deployed files, using AWS SDK for Java v2 and presigned URLs for direct browser upload/download.
- Docker Compose locally and on EC2. Build the frontend as a client-rendered SPA with Angular, Angular CLI, strict TypeScript, and Tailwind CSS; package its static files into the Spring Boot application. Use Angular CLI's supported build and development tooling, including its built-in Vite development server; do not add a standalone Vite setup. Angular CLI provides template and style HMR during development; application logic changes may require a full reload.
- Node is frontend build/development tooling only. Select and pin a supported Node version compatible with the approved Angular version.
- Terraform in one flat application root module: EC2 host, RDS, S3 app buckets, ECR, and security groups in the account's default VPC. No child modules, ECS, custom networking, or load balancer unless an approved plan explicitly allows them.
- GitHub Actions with one deployment job per approved environment, using GitHub Environments and OIDC.

### Required development container

- Commit `.devcontainer/devcontainer.json` at the project root, with its supporting Dockerfile and Compose development override. This is the primary development environment, separate from the minimal production runtime image.
- Install the approved Java JDK, Node/npm, Git, and tools needed by the Maven Wrapper in the development image. Pin compatible versions. Java 21 is the proposed application toolchain; select Node using Angular's official compatibility matrix and supported Node releases rather than copying an example Node 20 image. If JDTLS needs a newer Java runtime, install and configure it separately while retaining the approved application compilation target.
- Bind-mount the local repository into a documented container workspace. Source edits remain on the host and are immediately visible in the container. Run dependency installation, compilation, tests, Spring Boot development processes, Angular CLI, terminals, tasks, and language servers inside containers; do not require host Java, Maven, or Node installations.
- Install OpenAI Codex CLI in the development image using the official `@openai/codex` npm package with an explicitly pinned release (`npm install -g @openai/codex@<approved-version>`). Ensure `codex` is on the non-root development user's PATH. Include it only in the development image, not the production app/worker image.
- Run `codex` from the mounted repository in the container terminal so it reads the project's AGENTS.md, edits the same source files, and runs commands with the container's Java/Node toolchains. Codex model inference requires network access to OpenAI; the container hosts the CLI and its local tools, not the model itself.
- Document interactive authentication from inside the container. Prefer `codex login --device-auth` for ChatGPT login where the account/workspace enables device authentication, completing the browser step on the host. If unavailable, document a verified supported fallback. Authentication is performed at runtime, never during image build or automatic setup hooks.
- Persist the development user's `~/.codex` directory in a dedicated user-specific Docker volume outside the repository so login/configuration survive container rebuilds. Treat cached credentials as secrets, restrict permissions, and document logout and volume removal. Do not automatically mount the host's entire Codex directory or bake authentication state into images. If API-key authentication is selected, inject its secret only into the development container at runtime; do not pass it to app, worker, production, or build stages.
- Verify `codex --version`, workspace access, and the documented login workflow without printing credentials. A live model request is a separate authenticated check; report it as unverified unless performed. Keep the user's approval and sandbox settings explicit; do not automatically disable Codex safeguards.
- Use a non-root development user with writable workspace and cache directories. Keep container-specific `node_modules` and Maven caches in named volumes rather than sharing host-native dependencies. Document volume ownership and cleanup; dependency caches and build artifacts are not durable application data.
- Use Docker Compose to put the development container and Postgres on the same local network. Use service DNS names for database connections. Run the web and worker processes with distinct Spring profiles and explicit tasks; do not also start duplicate packaged app/worker services in the interactive development mode.
- Keep the existing full-stack `docker compose up` mode for packaged local testing. Provide a separate development override/profile for the editor workflow and document the exact commands for each. Production must exclude the development container, source bind mounts, caches, language servers, and Angular development server.
- Run Angular CLI's development server inside the development container with HMR, listening on the container interface. Forward the Angular and Spring Boot ports to the host browser and bind published development ports to loopback by default. Proxy `/api` to the backend, and preserve authentication cookies and CSRF behavior. Document watcher settings only if bind-mounted file changes require them.
- Install a pinned release of Coder's `code-server` in the same development image and run it as the non-root development user, opening the mounted repository. Start it automatically in browser-development mode, independently of app/worker tasks. The browser editor's terminal must expose the development container's Java/Node tools and Codex CLI.
- Reserve container port 5678 for code-server and publish it as `127.0.0.1:5678:5678`, accessible at `http://localhost:5678`. code-server listens on `0.0.0.0:5678` inside the container; the Docker host binding limits access to localhost. Use distinct backend and Angular ports (for example 8081 and 4200) and document them consistently. Do not expose code-server in production or on the public network.
- Keep code-server password authentication enabled. Supply a user-provided or generated `CODE_SERVER_PASSWORD` via gitignored local `.env`, mapping it to code-server's `PASSWORD` environment variable only in the development service. Include a placeholder in `.env.example`; never bake or print the password. Document login and configuration without displaying secret values.
- Preinstall the VSCodeVim extension (`vscodevim.vim`) for the development user using `code-server --install-extension vscodevim.vim`, pinning a verified extension version or verified release VSIX for reproducibility. Use the supported Open VSX distribution or the extension author's releases; do not assume Microsoft Marketplace access. Enable Vim motions by default and verify normal, insert, and visual modes in the browser. Document browser shortcut conflicts and any minimal required settings.
- Install verified code-server-compatible Java and Angular/TypeScript language extensions as part of setup. Persist code-server user settings and extension state in dedicated Docker volumes with correct ownership; seed pinned extensions idempotently so mounted volumes do not hide or bypass first-start installation.
- Document the browser workflow: start the development Compose mode, open localhost, authenticate, edit the mounted project, and run app/worker/frontend tasks and `codex` in the integrated terminal. A browser plus Docker is sufficient for this workflow. Verify startup, password protection, Vim motions, terminals, and persistence during implementation.
- Document Docker and a supported browser as host prerequisites for code-server, initial setup, opening the container, starting app/worker/frontend tasks, port access, and container restart/rebuild after configuration changes. Verify these behaviors with the selected versions.
- Read local configuration from the gitignored `.env` via explicit Compose configuration. Do not bake credentials into the development image, automatically mount host AWS credentials, or contact AWS by default. Do not mount the Docker socket by default; if Testcontainers requires Docker access, resolve and document that explicit development exception in the approved plan.
- During implementation, verify fresh container startup, Java/Node versions, dependency installation, backend/frontend tests, source-change detection, frontend HMR, database connectivity, and editor language-server startup. Report checks requiring interactive browser access as unverified unless actually observed.

### Processes and local development

- One Spring Boot web process serves REST JSON under `/api/v1`, authentication endpoints under `/api/auth/*`, `/healthz`, and frontend assets with an SPA fallback. Unknown API routes return API errors rather than the SPA. Limit SPA fallback to appropriate browser GET requests; never route API, auth, health, or missing asset requests to `index.html`.
- Run background jobs in a separate worker Compose service using the same image with a worker Spring profile and no HTTP server. Disable Quartz execution in the web process; the web process may submit durable jobs using the documented JDBC job-store integration.
- Register recurring Quartz jobs and triggers idempotently at worker startup. Use explicit time zones, bounded retries with backoff, misfire policies, failure status, and recovery procedures. Quartz does not automatically supply application retry semantics.
- Configure Quartz clustering with unique worker instance IDs so multiple workers coordinate through Postgres. Use job concurrency controls where needed, but also make side effects idempotent. Do not run business schedules through `@Scheduled`, timers, in-memory executors, or a separate always-on scheduler process.
- Keep application processes stateless and ready for multiple copies. Sessions live in Postgres. No persistent application state on local disk. Local transient memory is not a durable queue.
- `docker compose up` reads `.env` and starts the full local stack: app, worker, and Postgres. It must not require AWS credentials or contact AWS by default. Compose must explicitly pass required environment variables to the services; Spring Boot does not automatically read a repository `.env` file.
- Share app/worker service definitions between local and production Compose configurations. Use explicit profiles or overrides so production uses RDS and does not start a local Postgres service. Document both commands and migration ordering.
- The required development-container workflow uses Angular CLI's development server for HMR and proxies `/api` to Spring Boot. Document its command and proxy configuration. Production serves compiled assets directly from Spring Boot and does not run Angular CLI.
- By default, local S3-dependent features are disabled with a clear unavailable response/UI state. Do not silently store files on disk or add a storage emulator. If local upload testing is required, resolve and approve an explicit exception in the plan. Connecting local development to AWS requires the user's permission and an explicit override.

### Data and API conventions

- Flyway SQL migrations under `src/main/resources/db/migration` are the source of truth for database schema, including application, Spring Session, and Quartz tables. Use JPA mappings to represent the schema; configure Hibernate schema validation rather than automatic creation or updates.
- Disable automatic destructive schema initialization for session and Quartz tables. Manage their version-specific schemas through reviewed Flyway migrations; never recreate job tables on startup.
- Use request/response DTOs and Jakarta Bean Validation. Do not expose JPA entities directly through the API or accept arbitrary entity fields from clients. Request validation and SQL constraints are separate layers; keep both aligned and verify important invariants with integration tests.
- Validate every API input and enforce authorization, including record ownership and roles. Application API errors use `{ error: { code, message } }`, including authentication and access-denied errors. Centralize exception handling and avoid exposing stack traces, SQL, or credentials.
- Define signup, login, logout, session inspection, and CSRF-token endpoints under `/api/auth/*` in the approved plan. Spring Security supplies security primitives, not a complete signup/password-reset product; explicitly plan any verification, reset, or email flows.
- Hash passwords with a Spring Security password encoder using a supported adaptive algorithm. Store no plaintext passwords. Use secure, HttpOnly session cookies over HTTPS with an explicit SameSite policy, session expiration, and session fixation protection.
- Keep CSRF protection enabled for cookie-authenticated state-changing requests. Configure Angular's XSRF integration and Spring Security's token handling together, including refreshing tokens after login/logout. Permit only approved origins where CORS is needed. Worker processes do not expose authentication endpoints.
- Paginate lists with `?page=` and `?per_page=`; use a one-based public API and translate to Spring Data's zero-based pagination internally. Document defaults, limits, stable sorting, and response metadata.
- Index hot query paths, avoid N+1 queries, and batch related writes. Use service-layer transactions when related changes must succeed together. Disable Open Session in View; load and map required data within explicit transaction boundaries.
- Submit durable background work through Quartz; make retryable operations idempotent. Use an approved transactional outbox or verified transactionally coordinated submission when a business write and job submission must succeed together. Do not assume asynchronous execution is atomic with a database commit.
- Keep the JSON API stable and documented for a future iOS client. Document how a native client authenticates and handles sessions; do not introduce JWTs or a second auth architecture without approval.
- Supply database credentials, `APP_BASE_URL`, session/cookie policy, and other runtime configuration through local `.env` or environment-scoped deployment configuration. Identify which keys are secrets, non-secret settings, or provisioning outputs. Do not carry over `BETTER_AUTH_SECRET` or `BETTER_AUTH_URL`; Spring Session does not require a Better Auth signing secret.
- Inject deployment secrets into the host environment file or Compose at deploy time, never into the image. Use environment-specific Spring configuration without embedded credentials.
- Use plain, simple code. No code generator CLI. Angular CLI and Maven are allowed for build, serve, and test commands, but not code generation. Do not use Spring Initializr or generated scaffolding unless explicitly authorized.
- Use JUnit and Spring Boot testing support for backend tests. Use real Postgres integration tests, for example with Testcontainers, for persistence, migrations, session behavior, and job coordination where needed; do not substitute H2 for Postgres semantics.

## Phase 4 — Launch

Launch only when requested and after the target environment passes Phase 2. Routine authenticated AWS and deployment operations run in GitHub Actions using that environment's OIDC role. The only local bootstrap exception is the explicitly approved SSO procedure documented in Phase 2.

Encode the deployment in `.github/workflows/deploy.yml`, with one deployment job per approved environment:

1. Validate required configuration and secret presence without printing values; fail with a clear message naming missing keys or role configuration.
2. Install frontend dependencies, run appropriate frontend and backend tests with the pinned Node and Java toolchains, build the Angular frontend, and package the Spring Boot application with Maven.
3. Assume the environment's AWS role through OIDC and verify the expected account/role.
4. Run `terraform init`, review the planned changes, and apply the approved infrastructure. Resolve deployment outputs.
5. Build the image and push it to ECR.
6. Prepare the Compose bundle and runtime environment on EC2 using the transport specified in the plan. Ensure database credentials, the final `APP_BASE_URL`, secure session/cookie settings, and all required secrets are configured before startup.
7. Run Flyway migrations against RDS from the execution location specified in the plan before activating code that requires them. Prevent concurrent migration runs and stop rollout on failure. Use a dedicated migration execution mode/location specified in the plan; app and worker startup must not independently race to migrate. Include required session and Quartz tables.
8. Activate the new app/worker services and verify `/healthz`, signup/login, and creation of a record through the API. Use designated smoke-test data and accounts.

Document deployment commands, migration compatibility, recovery steps, and any manual bootstrap. A workflow rerun must reuse existing infrastructure safely. Do not use teardown as a recovery strategy.

## Phase 5 — Teardown

Teardown is destructive and may permanently delete data. Never perform it during launch, redeployment, or routine cleanup.

Only proceed after the user explicitly requests teardown and confirms the exact environment and resources to delete. Offer database and file backups before any deletion; record whether the user accepts or skips them. Backups must be stored outside resources being destroyed.

Encode teardown in `.github/workflows/teardown.yml` using `workflow_dispatch`, an environment-scoped OIDC job, an explicit environment confirmation input, and a backup choice. Fail before deletion if confirmation is absent or mismatched. A requested backup must complete successfully before deletion proceeds.

1. If selected, dump RDS with `pg_dump` and copy S3 objects to the approved backup destination.
2. Remove the app's DNS record, if applicable, using the approved DNS procedure.
3. Empty the target app's S3 buckets, including versions and delete markers where applicable. Do not empty unrelated or backup buckets.
4. Review the destroy scope, then run `terraform destroy` for the target environment to remove its application resources. Account explicitly for RDS deletion settings and ECR images.
5. Delete the state bucket only after confirming destroy completed, no other project or environment shares it, and its deletion is included in the user's confirmation.
6. Remove the target environment's app secrets and remove or disable its deployment role after the AWS cleanup is complete. If the workflow lacks the necessary GitHub or IAM permissions, provide the user with the exact remaining manual steps; do not claim cleanup is complete.
7. Verify that the target application's AWS resources are gone and its old URL no longer serves the application. Report any remaining backups, shared resources, or manual cleanup.

Local teardown with SSO is allowed only when explicitly confirmed and documented instead of the Actions procedure. Apply the same scope, backup, and confirmation requirements.
