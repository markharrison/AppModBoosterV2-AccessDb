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

### Database

Database: SQL Server

- Database seeding with sample data on startup (disabled in production) — done by the API App.

### Environments

Support both local development and cloud deployment.

Cloud environments are:

- `dev` (Development)
- `staging` (Staging)
- `prod` (Production)

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

- Use Bicep for Infrastructure as Code
- Compute: 1 × Azure Linux App Service Plan (default `B1 Basic` but overridable, `DOTNETCORE|10.0`)
  - All web applications share the same plan
- Database: 1 × Azure SQL Database (Basic tier)
- Location: default `eastus` but overridable
- Azure environments: Development | Staging | Production
- Azure resource naming: `{resourcetype}-{app}-{env}`
- Use Application Insights for telemetry

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
- These ports must be set in `launchSettings.json`, `appsettings.Development.json`, run-local script, and documentation. No file should use different ports.

### Infrastructure

- Use only `dev`, `staging`, and `prod` for all environment names everywhere, including GitHub environments and OIDC subjects.

### Scripts

- Ensure PowerShell scripts are robust — they must be idempotent and safe to rerun after partial completion.

### GitHub Workflows

- Use only currently supported GitHub Action versions.
- Do not use Actions that depend on deprecated Node.js runtimes.
- Validate that the final workflow YAML is valid before finishing.




