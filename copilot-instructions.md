# Copilot Instructions

This file defines the target requirements for the Booster modernization pipeline.
It is automatically injected into all GitHub Copilot interactions in this repository.

All Booster pipeline agents derive their architectural and technology decisions from
this file. **Edit this file to change how the modernized application is built.**

---

## Target Application

The legacy application to modernize is located in `./input/`.
Use the `Booster` agent to analyze and modernize it.

---

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌────────────────────┐
│   Web App   │────>│   API App   │────>│     Database       │
│  (Razor UI) │     │  (Web API)  │     │ (SQLite/SQLServer) │
└─────────────┘     └─────────────┘     └────────────────────┘
```

Two web applications:

- **Web App** — User Interface (Razor). Calls the API App server-side using HttpClient. Never connects to the database directly. The browser only communicates with the Web App.
- **API App** — REST APIs. Provides the Database Context and migrations. Not directly accessed by browsers.

CORS is not required as the API App is only called server-side by the Web App.

### Environments

Support both local development and cloud deployment.

Local developer environent:

- `local` (dev test on developer machines)

Cloud environments are:

- `dev` (Development)
- `staging` (Staging)
- `prod` (Production)

### Database

Database: SQL Server

- Database seeding with sample data (for environments: `local` and `dev`) — is done by the API App on startup

---

## Tech Stack

### Application

- Language: C#
- Frameworks: .NET 10, ASP.NET Core, EF Core, Razor views
- EF Core to support code-first migrations

#### Web App

- HTML / CSS / JavaScript
- Bootstrap (CSS / JS) Framework

#### API App

- Controller-based REST APIs
- Swagger UI (disabled in production)

### Infrastructure

Must support both local development and Azure cloud deployment.

#### Local Development

- Database: SQLite

#### Cloud Deployment

- Use "AZ scripts" for Infrastructure as Code
- Deployment scripts should be designed to be run locally or via GitHub Workflows

- Compute: 1 × Azure Linux App Service Plan (default `B1` but overridable)
  - All web applications share the same plan
- Database: 1 × Azure SQL Database (Basic tier)
- Location: default `northeurope` but overridable
- Azure environments: Development | Staging | Production ... allow scripts to deploy environments independently
- Azure resource naming: `{resourcetype}-{app}-{env}`
- Use Application Insights for telemetry for `prod` environment only

### Developer Platform

- GitHub Repos for storing code, documents, etc.
- GitHub Workflows/Actions for CI/CD

### Authentication

#### Local Development

- No authentication between Web App and API App
- Uses SQLite; no SQL authentication required

#### Cloud Deployment

- Authentication between Web App and API App using managed identity (Microsoft Entra ID)
- API App authenticates to Azure SQL Database using managed identity (Microsoft Entra ID)

#### CI/CD

- GitHub Workflows with OIDC federated credentials (`azure/login` with `client-id`, `tenant-id`, `subscription-id`)
- Managed Identity — no `AZURE_CREDENTIALS` secret

### Testing

- NUnit (test framework)
- Playwright (end-to-end browser testing)

### Scripts

All scripts must be PowerShell — do not use Bash.

---

## Policy Constraints (MCAPS)

- Azure AD-only authentication on SQL Server — no SQL auth (`azureADOnlyAuthentication: true`, no `administratorLogin`/`administratorLoginPassword`)
- Managed identities for all service-to-service auth — no plaintext credentials or secrets
- Connection strings use `Authentication=Active Directory Managed Identity`
- TLS 1.2 minimum, HTTPS-only on web apps

---

## Implementation Instructions

### Application

- With CDN `<script>` or `<link>` tags, do NOT include integrity attributes. Incorrect hashes will cause browsers to block the resource.
- The Web App must know the URL of the API App. In local dev mode, use the following ports:
  - Web App runs on port **5200**
  - API App runs on port **5201**
- These ports must be set in `launchSettings.json`, `appsettings.local.json`,`appsettings.dev.json` , run-local script, and documentation. No file should use different ports.

### Infrastructure

- Use only `dev`, `staging`, and `prod` for all environment names everywhere, including GitHub environments and OIDC subjects.
- The canonical application name prefix is `northwind`. This value must be the default for every `$AppName` / `app-name` parameter across all scripts, workflows, and OIDC setup. Never use a different default.
- Azure resource naming follows `{resourcetype}-{app}-{env}` (e.g., `app-northwind-api-dev`, `sql-northwind-dev`, `sqldb-northwind-dev`). All appsettings, connection strings, and API base URLs must use the same naming convention — no hardcoded names that differ from the convention.
- Azure App Service terminates SSL at the load balancer. Apps receive plain HTTP on port 8080 internally. All apps must check the `DisableHttpsRedirect` configuration setting (set to `"true"` in Azure) and skip `UseHttpsRedirection()` when it is `"true"`. Do NOT use environment-name checks (e.g., `IsEnvironment("local")`) to control HTTPS redirect — use the config setting so it works correctly in all environments.

### Application Resilience

- Database migrations and seeding run on API App startup. These must include retry logic with backoff (at least 5 attempts) to handle cloud cold-start timing where the SQL Database or managed identity grant may not be immediately available.
- Connection strings in `appsettings.{env}.json` are overridden by Azure App Settings at runtime, but they must still use the correct naming convention as documentation and fallback — never use placeholder or legacy names.

### Scripts

- Ensure PowerShell scripts are robust — they must be idempotent and safe to rerun after partial completion.
- Every script and workflow that accepts an `AppName` / `app-name` parameter must default to `northwind`. Verify consistency across all files before finishing.

### GitHub Workflows

- Use only currently supported GitHub Action versions.
- Do not use Actions that depend on deprecated Node.js runtimes.
- Validate that the final workflow YAML is valid before finishing.
- Verify that all default parameter values in workflows match the defaults in the corresponding PowerShell scripts they invoke.
