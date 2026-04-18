# Trainer-Demo-Deploy - Azure Demo Builder

An agent-driven workflow for building, deploying, and demonstrating Azure infrastructure scenarios — powered by GitHub Copilot custom agents and Azure Developer CLI (`azd`).

## What Is This?

[Trainer-Demo-Deploy](https://aka.ms/trainer-demo-deploy) Azure Demo Builder automates the end-to-end lifecycle of Azure demo environments. Describe the Azure scenario you want to build in natural language, and a pipeline of specialized AI agents handles requirements gathering, architecture design, Bicep code generation, deployment, and demo guide creation — all within VS Code.

Instead of manually writing Bicep templates, configuring `azd`, and preparing demo scripts, you interact with a single **Conductor** agent that coordinates specialized agents through the full workflow.

## How It Works

The workflow is a seven-step pipeline with one conditional step. Each step is handled by a dedicated agent that produces versioned artifacts in `scenario/{project}/`. The Conductor coordinates handoffs automatically.

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 02. Valida-  │───▶│ 03. Archi-   │───▶│ 04. Diagram- │
│    tions     │    │    tect      │    │    mer       │
└──────────────┘    └──────────────┘    └──────────────┘
                                              │
       ┌──────────────────────────────────────┘
       ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 05. Bicep    │───▶│ 06. Deploy   │───▶│ 07. Demo     │───▶│ 08. Contri-  │
│  (IaC Gen)   │    │  (azd up)    │    │    Guide     │    │    bute      │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
       │
       ▼
┌──────────────┐
│ 05b. Dev     │
│  (webapp)    │
└──────────────┘
```

> The **01-conductor** agent conducts the full pipeline — it is not shown as a step because it delegates to each agent below.

| Step | Agent            | What It Does                                                       | Key Output                                            |
| ---- | ---------------- | ------------------------------------------------------------------ | ----------------------------------------------------- |
| 02   | **Validations**  | Parses your scenario description into structured requirements      | `01-requirements.md`                                  |
| 03   | **Architect**    | Recommends Azure services, SKUs, and documents trade-offs          | `02-architecture-assessment.md`                       |
| 04   | **Diagrammer**   | Generates Python-based architecture diagrams and ADRs              | `03-architect-diagram.py`, `03-architect-diagram.png` |
| 05   | **Bicep**        | Runs governance discovery, generates AVM-first Bicep templates     | `infra/main.bicep`, `04-implementation-plan.md`       |
| 05b  | **Development**  | Scaffolds a .NET 10 sample webapp with industry seed data          | `src/`, `07-webapp-summary.md`                        |
| 06   | **Deploy**       | Runs `azd up` with what-if analysis and validates deployment       | `06-deployment-summary.md`                            |
| 07   | **DemoGuide**    | Produces an audience-aware demo runbook with talking points        | `demoguide/demoguide.md`                              |
| 08   | **Contribute**   | Forks, branches, commits, and opens a draft PR for the scenario    | Draft PR + optional GitHub Issue                      |

> Step 05b is conditional — it runs only when the architecture includes a compute host (App Service, Container Apps, etc.) that needs a sample app. Step 08 is user-invoked after the workflow completes.

All artifacts land in `scenario/{project}/`, giving you a self-contained, version-controlled demo package.

## Prerequisites

| Requirement                                                                                                     | Purpose                                   |
| --------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| [VS Code](https://code.visualstudio.com/) (1.100+)                                                              | IDE with agent support                    |
| [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat) (active subscription) | Powers the AI agents                      |
| [Azure Developer CLI (`azd`)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)      | Deploys infrastructure                    |
| [Azure CLI (`az`)](https://learn.microsoft.com/cli/azure/install-azure-cli)                                     | Governance discovery and authentication   |
| [Bicep CLI](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install)                             | Template compilation and linting          |
| [Python 3.10+](https://www.python.org/)                                                                         | Diagram generation                        |
| [Graphviz](https://graphviz.org/download/)                                                                      | Required by the `diagrams` Python library |
| [GitHub CLI (`gh`)](https://cli.github.com/)                                                                     | Fork, branch, PR, and issue operations (required for contributing) |
| An Azure subscription                                                                                           | Target for deployments                    |

### Recommended VS Code Extensions

The workspace includes an [extensions.json](.vscode/extensions.json) with recommendations. Key ones:

- `ms-azuretools.vscode-azure-mcp-server` — Azure MCP server for resource queries
- `ms-azuretools.vscode-bicep` — Bicep language support
- `ms-vscode.copilot-mermaid-diagram` — Mermaid diagram rendering in chat

### Python Dependencies

```bash
pip install -r requirements.txt
```

## Getting Started

### 1. Clone and Open

```bash
git clone https://github.com/petender/tdd-azd-demo-builder.git
cd tdd-azd-demo-builder
code .
```

### 2. Sign In to Azure

```bash
az login
azd auth login
```

### 3. Install Python Dependencies

```bash
pip install -r requirements.txt
```

### 4. Open Copilot Chat and Run

Open the Copilot Chat panel (`Ctrl+Shift+I`), select the **01-conductor** agent, and describe your scenario. The Conductor handles everything from there.

#### Example Prompt

> I want to build an Azure demo scenario with a web app hosted on Azure App Service, connected to an Azure SQL Database, with secrets stored in Key Vault, and monitoring via Application Insights. The web app should use managed identity to access both the database and Key Vault. Include a VNet with private endpoints for the SQL database and Key Vault.

The agent will:

1. Parse your description into structured requirements
2. Assess the architecture (services, SKUs, trade-offs)
3. Generate architecture diagrams
4. Produce governance-aware Bicep templates using Azure Verified Modules
5. Scaffold a sample webapp (if applicable)
6. Deploy to Azure via `azd up`
7. Create a step-by-step demo guide

#### More Example Prompts

**Hub-spoke network with VMs and Bastion:**

> Build a demo with a web VM and SQL VM in separate subnets, connected through Azure Firewall, with Bastion for secure access and Key Vault for password management.

**Containerized microservices:**

> Create an Azure Container Apps demo with three microservices, an Azure Container Registry, Application Insights for monitoring, and a Service Bus for async messaging between services.

**Serverless event-driven:**

> I need a serverless demo with Azure Functions triggered by Event Grid, writing to Cosmos DB, with API Management as the front door and Application Insights for observability.

## Project Structure

```
.github/
├── agents/                    # Agent definitions (one per workflow step)
│   ├── 01-conductor.agent.md
│   ├── 02-validation.agent.md
│   ├── 03-architect.agent.md
│   ├── 04-diagrammer.agent.md
│   ├── 05-bicep.agent.md
│   ├── 05b-development.agent.md
│   ├── 06-deploy.agent.md
│   ├── 07-demoguide.agent.md
│   └── 08-contribute.agent.md
├── instructions/              # File-type coding standards (Bicep, Markdown, Python, etc.)
├── PULL_REQUEST_TEMPLATE/     # PR template for scenario contributions
├── ISSUE_TEMPLATE/            # Issue template for new scenarios
└── skills/                    # Reusable knowledge consumed by agents
    ├── SKILL.md               # Consolidated skill (defaults, AVM, patterns, diagrams)
    ├── azure-artifacts/       # Artifact templates
    ├── azure-deploy/          # Deployment patterns
    ├── azure-diagrams/        # Diagram generation guides
    ├── azure-validate/        # Pre/post deployment validation
    └── webapp-development/    # .NET 10 sample webapp scaffolding patterns

scenario/                      # Generated demo projects (one folder per scenario)
├── api-storage-table-crud/    # Example: App Service + Storage Table API
│   ├── infra/
│   │   ├── main.bicep
│   │   ├── main.bicepparam
│   │   └── modules/
│   ├── src/                   # Sample .NET 10 webapp (when generated)
│   ├── demoguide/
│   ├── azure.yaml
│   └── README.md
├── azure-container-scenarios/ # Example: Container Apps + ACI + ACR
├── sentinel-threat-detection/ # Example: Sentinel + Log Analytics
└── ...
```

## Key Design Decisions

- **AVM-first**: The Bicep agent always uses [Azure Verified Modules](https://aka.ms/avm) when available, falling back to raw Bicep only when no AVM module exists.
- **Deployer data plane access**: When RBAC-enabled resources are deployed (Key Vault, Storage, etc.), the deploying user automatically receives data plane role assignments so they can immediately interact with the resources.
- **Convention over configuration**: Default region (`eastus2`), naming conventions (CAF-aligned), security baseline (TLS 1.2, HTTPS-only, managed identity), and tagging are baked into the skills and instructions.
- **Scoped contributions**: Only `infra/`, `demoguide/`, `azure.yaml`, and `README.md` are committed to upstream PRs. Working artifacts (requirements, architecture assessments, diagrams, src/) remain in the contributor's fork.

## Teardown

To remove all deployed resources for a scenario:

```bash
cd scenario/{project-name}
azd down --force --purge
```

## Understanding the Agent Architecture

New to GitHub Copilot custom agents? See
[AGENTSEXPLAINED.md](AGENTSEXPLAINED.md) for a technical deep-dive into how
agents, skills, instructions, and prompts work together — including how to
add your own agents, skills, and coding standards to the workflow.

## Contributing

We welcome scenario contributions from the community! There are two ways to contribute.

### Prerequisites for Contributing

| Requirement | Purpose |
|---|---|
| [GitHub CLI (`gh`)](https://cli.github.com/) | **Required.** The Contribute agent uses `gh` to fork the repo, create branches, open PRs, and file issues. Install and authenticate with `gh auth login` before contributing. |
| Completed scenario (Steps 1–6) | Your scenario folder under `scenario/` must contain the generated artifacts before contributing. |

> **Why `gh`?** The contribution workflow needs to fork the upstream repo, push to your fork, and open cross-fork pull requests. The GitHub CLI handles all of this with a single authenticated session — no manual token configuration required.

### Agent-Assisted (Recommended)

After completing the agent workflow (Steps 1–6), invoke the **08-Contribute** agent
in Copilot Chat. It will:

1. **Validate** your scenario artifacts for completeness
2. **Fork** the repo via `gh repo fork` (idempotent — safe to run repeatedly)
3. **Create a branch** named `contribute/{project-name}` on your fork
4. **Stage and commit** only `infra/`, `demoguide/`, `azure.yaml`, and `README.md` with a conventional commit
5. **Open a draft PR** against the upstream `main` branch using the Scenario Contribution template
6. **Optionally create a tracking GitHub Issue** for maintainer visibility

> Other scenario files (requirements, architecture assessments, diagrams, implementation plans, src/) remain in your fork for reference but are not included in the PR.

The agent performs a sensitive-data check before committing — it will refuse to proceed if `.azure/`, `.env`, `bin/`, `obj/`, `publish/`, or `applogs/` files would be staged.

### Manual

1. Install and authenticate the [GitHub CLI](https://cli.github.com/): `gh auth login`
2. Fork the repo and create a branch: `contribute/{your-scenario-name}`
3. Add only the following artifacts under `scenario/{project}/`:
   - `infra/main.bicep` + modules, `azure.yaml`, `README.md`
   - `demoguide/demoguide.md` + screenshots (recommended)
4. Ensure no sensitive files are included (`.azure/`, `.env`, `bin/`, `obj/`)
5. Open a draft PR using the **Scenario Contribution** template

### Extending the Agents

1. **Agents** live in `.github/agents/` — each file defines one workflow step
2. **Skills** live in `.github/skills/` — shared knowledge that agents reference
3. **Instructions** live in `.github/instructions/` — file-type specific coding standards

To modify agent behavior, edit the corresponding `.agent.md` file. To change
patterns or defaults that apply across agents, update `SKILL.md` or the
relevant instruction file.

## License

MIT
