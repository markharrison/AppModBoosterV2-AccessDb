# AppModBoosterV2

AppModBoosterV2 is a GitHub Copilot agent workflow for modernizing a legacy database application into a cloud-ready web application.

The repo is centered on one orchestrator agent and six specialist subagents defined in `.github/agents/`.

## Agents

- `Booster` - orchestrates the end-to-end modernization pipeline.
- `Booster Analyst Agent` - analyzes the legacy application and documents what exists.
- `Booster Spec Agent` - turns the analysis into an implementation-ready specification.
- `Booster APIDev Agent` - generates the API application and unit tests.
- `Booster WebDev Agent` - generates the web application and end-to-end tests.
- `Booster IaC Agent` - generates infrastructure, workflows, and setup scripts.
- `Booster Tech Writer Agent` - generates project documentation.

## Workflow

The pipeline runs in this order:

1. Analyst
2. Spec
3. APIDev
4. WebDev
5. IaC
6. Tech Writer

The orchestrator definition is in `.github/agents/booster.agent.md`.
The specialist subagents are in `.github/agents/_subagents/`.

## Repository Layout

### Requirements & Inputs

- `copilot-instructions.md` - target requirements (architecture, tech stack, policy constraints, implementation instructions). Edit this file to change how the modernized application is built.

- `input/` - source legacy application files.

### Outputs

- `output/src/` - generated application source projects.
- `output/infra/` - infrastructure definitions.
- `output/scripts/` - setup and helper scripts.
- `output/test/` - automated tests.
- `output/state/` - step tracking for the modernization workflow.
- `.github/workflows/` - generated CI/CD workflow files.

## How To Use

Invoke the `Booster` agent and describe the legacy application you want to modernize.

Example:

```text
Modernize the legacy database app in ./input/
```

## Example

Modernise an Access Database application to a Web App/API App hosted on Azure App Service and using Azure SQL Database.
