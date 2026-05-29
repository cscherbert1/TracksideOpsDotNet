# TracksideOps — Phase 1 Architecture

This document records the initial technical architecture and major technology choices for Phase 1 of TracksideOps. It is intended as a short, actionable reference for developers and automation agents.

## Project goal (Phase 1)

Implement a CRUD web application for model-railroading domain objects such as Locomotives, Rolling Stock (Cars), and related entities. Phase 1 focuses on developer productivity, local development experience, and establishing core infrastructure (database, ORM, testing, and basic UI patterns).

## Major technology decisions

- .NET: Continue using .NET 10 (project currently targets net10.0).
  - Rationale: the repository and projects are already targeting .NET 10; keeping it avoids upgrade work and keeps dependencies aligned.

- Application type: ASP.NET Core Razor Pages
  - Rationale: Razor Pages provides page-focused patterns ideal for CRUD scenarios and keeps UI logic close to page views. It simplifies development for small teams and rapid iteration.

- Razor Pages structure: feature-based folders
  - Pattern: Organize Pages by domain feature. Example:
	- Pages/Locomotives/Index.cshtml
	- Pages/Locomotives/Create.cshtml
	- Pages/Locomotives/Edit.cshtml
	- Pages/Cars/Index.cshtml
  - Rationale: feature-based organization keeps related UI, page models, and view files together and scales well as features are added.

- Database: PostgreSQL (local development via Docker)
  - Rationale: PostgreSQL is reliable, open-source, and well-supported. Using Docker for development ensures every developer has a reproducible database.

- ORM: Entity Framework Core, Code-First with Migrations
  - Rationale: Code-first development with migrations enables iterative model changes and keeps schema versioning in source control. Place DbContext and migrations in the Infrastructure project (TracksideOps.Infrastructure).

- Testing: xUnit for unit tests
  - Rationale: The repo already uses xUnit; we will continue with it for unit testing. Integration tests against Postgres can be introduced later (e.g., with Testcontainers) during Phase 2.

- Docker usage (development):
  - Use docker-compose for local development to run Postgres and optionally the web app.
  - Provide a multi-stage Dockerfile template for building small production images (useful for future CI/CD).
  - Rationale: docker-compose is the simplest approach for local orchestration; multi-stage Dockerfiles reduce image size for production images.

- Repository structure: keep the current multi-project layout
  - Rationale: Current layout separates domain, infrastructure, web, and tests which is appropriate for maintainability. Add an `ai/phase1` folder for architecture and agent artifacts.

- Initial deployment/local development: focus on local development only in Phase 1
  - Workflow documented below for running Postgres in Docker, applying EF Core migrations, running the app, and running tests.

## Developer setup and local workflow

Prerequisites
- .NET 10 SDK
- Docker Desktop (or other Docker runtime)
- Visual Studio 2022/2026 or compatible editor

1. Start Postgres for development

Create a `docker-compose.yml` in the repo root or run the provided snippet in this document. Example compose service:

```yaml
version: '3.8'
services:
  db:
	image: postgres:15
	environment:
	  POSTGRES_USER: trackside
	  POSTGRES_PASSWORD: changeme
	  POSTGRES_DB: trackside_dev
	ports:
	  - "5432:5432"
	volumes:
	  - trackside_pgdata:/var/lib/postgresql/data

volumes:
  trackside_pgdata:
```

Start it:

- docker compose up -d

2. Configure the connection string

Add or update `appsettings.Development.json` in the TracksideOps.Web project with:

```json
{
  "ConnectionStrings": {
	"DefaultConnection": "Host=localhost;Port=5432;Database=trackside_dev;Username=trackside;Password=changeme"
  }
}
```

3. Create and apply EF Core migrations

- Add migration (from repo root):

  dotnet ef migrations add InitialCreate --project TracksideOps.Infrastructure --startup-project TracksideOps.Web

- Apply migrations to local DB:

  dotnet ef database update --project TracksideOps.Infrastructure --startup-project TracksideOps.Web

4. Run the app

- From Visual Studio: set TracksideOps.Web as startup and run.
- Or: dotnet run --project TracksideOps.Web

5. Run tests

- dotnet test

## Dockerfile and docker-compose examples

Minimal multi-stage Dockerfile (for future production images):

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . ./
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish ./
ENTRYPOINT ["dotnet", "TracksideOps.Web.dll"]
```

Note: The Dockerfile uses .NET 8 images as an example for compatibility; when building images for .NET 10 you should use the corresponding .NET 10 base images (update image tags when available).

## EF Core notes

- Keep DbContext and entity configurations in TracksideOps.Infrastructure.
- Place migrations in the same project as DbContext to keep schema changes collocated with infrastructure code.
- Use explicit migration names and keep migrations small and focused.

## Testing guidance

- Unit tests: xUnit in TracksideOps.Tests. Keep tests fast and isolated from external resources.
- Integration tests (future): Use Testcontainers or a lightweight docker-compose orchestration to run Postgres for test runs; keep integration tests in a separate test project (TracksideOps.IntegrationTests) and run on demand.

## Next steps for Phase 1

1. Implement DbContext and initial entities (Locomotive, Car) in TracksideOps.Infrastructure.
2. Add initial migration and seed sample data for local development.
3. Create Razor Pages per feature (e.g., Pages/Locomotives) with CRUD scaffolding.
4. Add basic validation and unit tests for domain logic.
5. Add developer docs and update this ARCHITECTURE.md with any deviations or refinements.

## Risks & Future work

- Production deployment strategy (Azure App Service, AKS, container registry, secrets management) is deferred to Phase 2.
- Integration tests and CI/CD pipelines are out of scope for Phase 1 but recommended for Phase 2.

---

Document location: `ai/phase1/ARCHITECTURE.md`

If you want edits or additional details (e.g., sample seed data, exact Dockerfile targeting .NET 10 images, or CI/CD starter config), tell me which to include and I'll update the document.
