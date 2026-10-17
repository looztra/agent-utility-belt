---
name: document-architecture
description: Analyze an existing application and create or update a concise, evidence-based Markdown architecture document set. Use this skill whenever the user asks for architecture documentation, application structure documentation, system design documentation, external-service inventory, endpoint/dependency diagrams, data-model documentation, or an architecture overview derived from a codebase. Inspect the current implementation first, preserve existing documentation, and clearly distinguish confirmed facts from unknowns.
compatibility: Requires read access to the application source tree and permission to create or update Markdown files in the requested documentation directory.
---

# Document application architecture

Analyze the current application and document how it is actually organized and architected. The documentation must describe the implementation found in the repository, not an assumed or idealized design.

## Scope and output location

### Application scope

Before writing, determine which application(s) to document. A repository may contain several independently runnable applications (for example a web app, a worker, and an SDK in a monorepo). Detect them from workspace manifests, per-directory build files, Dockerfiles, deployment definitions, and entry points.

- If the user named a target application, document only that one.
- If the repository contains a single application, document it.
- If it contains several and the user did not name a target, ask the user whether to document one of them or all of them. Never merge separate applications into one document set.
- When documenting several applications, produce one document set per application (for example `docs/architecture/<app-name>/`), and add a top-level `README.md` index linking to each set and summarizing how the applications interact.

### Output location

Resolve the output location in this order:

1. A location the user specifies. Treat a user-provided path as repository-relative unless they explicitly provide an absolute path.
2. Existing architecture documentation for the target application, found during investigation (for example `docs/architecture.md`, `docs/system-design.md`, or `handbook/architecture/`). Update it in place instead of creating a parallel set. If its layout differs from the five-file set below, ask the user whether to restructure it into that layout or update it in its current structure; do not silently leave a stale copy next to a new set.
3. `docs/architecture` (or `docs/architecture/<app-name>/` per application) when no existing architecture documentation is found.

Create or update these files in the output directory:

- `README.md`
- `01-architecture-overview.md`
- `02-external-services.md`
- `03-endpoints-and-dependencies.md`
- `04-data-models.md`

If the directory or files already exist, update them to reflect the current code. Preserve useful existing content and add or revise sections rather than deleting documentation wholesale. Remove or correct statements only when repository evidence shows they are obsolete. Do not modify unrelated documentation.

## Investigation workflow

1. Inspect the repository structure, README files, build and dependency manifests, configuration, source directories, tests, migrations, schemas, infrastructure definitions, and existing architecture documentation. Resolve the application scope and output location (see above) before writing anything.
2. Identify the application entry points, runtime components, module/package boundaries, primary control flow, persistence approach, asynchronous messaging, authentication boundaries, and core architectural principles. Cite concrete paths and symbols where useful.
3. Search for outbound integrations and infrastructure dependencies, including HTTP clients, SDKs, databases, caches, object storage, queues, Kafka topics, webhooks, identity providers, and observability services. Distinguish configured-but-unused services from services called by the code.
4. Inventory HTTP endpoints and trace their meaningful dependencies. Include database reads/writes, outbound HTTP calls, Kafka ingestion/emission, queues, caches, and other observable side effects. If the application has no HTTP endpoints, state that explicitly.
5. Identify persisted data models from ORM entities, schemas, migrations, repositories, serializers, and database configuration. Describe relationships only when supported by code or schema evidence.
6. Update all five files as one coherent documentation set. Prefer concise sections and tables over repetition. Include source paths so readers can verify important claims.
7. Check every Markdown file for valid fenced-code languages and required blank lines around headings and lists. Ensure Mermaid diagrams use fenced `mermaid` blocks and remain syntactically coherent.

When evidence is incomplete, say `Unknown`, `Not found in the inspected source`, or equivalent and name what was inspected. Never invent endpoints, integrations, data fields, relationships, or runtime behavior. Avoid including secrets, tokens, credentials, or sensitive configuration values; document only service names, purposes, and safe configuration key names.

## Required file contents

### `README.md`

Make this a very concise index. Link to the four numbered documents using relative links and include one short sentence explaining that the set is generated from the current application implementation.

### `01-architecture-overview.md`

Document:

- scope and inspected areas
- a high-level architecture summary
- runtime components and responsibilities
- module/package organization
- core principles and notable patterns
- request, event, or job flow where applicable
- deployment/runtime assumptions that are evidenced by the repository
- a short list of source references

Do not present conventions or inferred intent as fact unless the code supports them.

### `02-external-services.md`

Inventory each external service in a table or clearly separated sections. For every service, include its name, category, purpose, how the application uses it, relevant safe configuration keys, and evidence/source paths. Cover databases, HTTP APIs, identity providers, storage, messaging, caches, monitoring, and other non-local dependencies. State explicitly when no external service is found.

### `03-endpoints-and-dependencies.md`

Document available HTTP endpoints, grouped by component or resource when useful. Include method, path, purpose, authentication/authorization behavior when evidenced, and important downstream dependencies. Add a Mermaid flowchart showing endpoint-to-component-to-dependency relationships. Label dependency types such as database, HTTP API, Kafka ingestion, Kafka emission, queue, cache, or filesystem. If there are no HTTP endpoints, explain what interface is used instead and provide the most relevant dependency diagram.

### `04-data-models.md`

Describe persisted data models and associated request, response, event, or domain models where relevant. Include fields and their meaning only when supported by schemas or code, relationships, identifiers, lifecycle constraints, and source paths. Add a Mermaid `erDiagram` when relationships are applicable; otherwise explain why it is omitted. Separate database-persisted models from transient/API models.

## Markdown quality rules

Apply these rules to every generated or updated Markdown file:

- Put a blank line before and after every heading.
- Put a blank line before and after every list.
- Specify a language on every fenced code block. Use `mermaid` for Mermaid diagrams and an appropriate language for examples.
- Use valid relative links within the documentation set.
- Keep Mermaid identifiers simple and quote labels containing punctuation when needed.
- Keep the README concise and do not duplicate the detailed documents.
- Preserve Markdown content that remains accurate, but update stale claims and diagrams.

Before finishing, review the diff for accidental deletion, unsupported claims, secret leakage, broken links, malformed Mermaid fences, and documentation that no longer matches the current implementation.
