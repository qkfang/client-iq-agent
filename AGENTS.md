# Repository Agent Guide

This file is the operational guide for coding agents working in this repository. Follow more specific instructions under `.github/instructions/` when their `applyTo` pattern matches a file.

## Working Rules

- Keep logs and change descriptions concise.
- Prefer the smallest readable change that fixes the controlling code path.
- Do not add unit tests unless the user explicitly requests them.
- Do not edit Markdown files unless the user explicitly requests documentation changes.
- Never add customer names, tenant details, credentials, tokens, or sensitive information to code, logs, fixtures, or mock data.
- Never print secrets. Use environment variables, managed identity, workload identity, or interactive user authentication.
- Preserve unrelated worktree changes. Stage and commit explicit file paths only.
- Do not create branches or commits unless requested.
- Use `apply_patch` for manual edits to existing text files.
- Run the narrowest executable validation immediately after the first code edit.
- For Azure work, load and follow the available Azure best-practices tool or skill before planning or changing resources.
- For Fabric workspace files under `src/fabric/fabric_workspace/`, read `.github/instructions/fabric-workspace.instructions.md` first.
- For deployment Python under `infra/scripts/`, read `.github/instructions/infra-scripts.instructions.md` first.
- When editing `docs/DeploymentGuide.md`, read `.github/instructions/deployment-guide.instructions.md` first.

## Repository Map

- `azure.yaml`: azd service definitions and lifecycle hooks.
- `infra/main.bicep`: resource-group deployment entry point.
- `infra/main.parameters.json`: maps azd environment values into Bicep parameters.
- `infra/deploy_*.bicep`: resource-specific modules.
- `infra/scripts/install_microsoft_iq_solution.py`: postprovision orchestrator for Search, Foundry, Fabric, and optional hosted agents.
- `infra/scripts/remove_microsoft_iq_solution.py`: predown cleanup orchestrator.
- `infra/scripts/common/`: shared configuration, environment, logging, and utility code.
- `infra/scripts/foundry/`: Search, knowledge-base, model, and agent setup.
- `infra/scripts/fabric/`: Fabric capacity, workspace, notebook, job, and role operations.
- `infra/fabric/deploy/fabric_solution_installer.ipynb`: Fabric-hosted deployment notebook.
- `infra/fabric/data/`: source data copied into OneLake.
- `src/fabric/fabric_workspace/`: Fabric Git-format source for lakehouse, notebooks, semantic models, reports, ontologies, and data agents.
- `src/web/`: .NET web application deployed by azd as service `web`.
- `src/func/`: .NET isolated Function App deployed by azd as service `func`.
- `src/foundry/data_supplier/documents/`: documents indexed into Azure AI Search.
- `src/hosted*`: optional hosted-agent implementations and packaging.
- `docs/DeploymentGuide.md`: supported deployment and operator documentation.

## Deployment Model

Deployment has two independent phases. Never treat phase-one success as complete deployment.

### Phase 1: Azure Resources

`azd provision` deploys `infra/main.bicep`. It creates or connects the resource group, managed identities, storage, Search, Foundry account/project, model deployments, monitoring, messaging, container registry, Fabric capacity, App Service plan, web app, and Function App.

Important region controls:

- `AZURE_LOCATION`: default resource location.
- `AZURE_AI_DEPLOYMENTS_LOCATION`: Foundry and model location; required and model-dependent.
- `AZURE_SEARCH_SERVICE_LOCATION`: independent Search location.
- `AZURE_APP_SERVICE_LOCATION`: independent App Service and Function location.
- `AZURE_EXISTING_FABRIC_CAPACITY_NAME`: use an existing Fabric capacity instead of creating one.
- `AZURE_FABRIC_CAPACITY_SKU_NAME`: Fabric SKU, normally `F2` unless requirements say otherwise.

Region availability and quota differ by subscription and can change. Check before deployment. Do not assume one region supports Search capacity, Fabric quota, App Service quota, and the selected model simultaneously.

### Phase 2: Solution Population

The `azure.yaml` postprovision hook runs:

```text
infra/scripts/install_microsoft_iq_solution.py
```

The installer:

1. Populates Azure AI Search indexes and uploads document chunks.
2. Creates Search knowledge sources and knowledge bases.
3. Creates Foundry agents.
4. Creates or finds the Fabric workspace and assigns capacity.
5. Configures workspace administrators.
6. Uploads the Fabric installer notebook.
7. Runs the notebook to deploy lakehouse, notebooks, semantic models, reports, data, ontologies, and data agents.

The installer must call `load_all_env()` so direct execution reads the active `.azure/<environment>/.env`, project `.env`, and `infra/.env` values.

## Prerequisites

Before any deployment:

1. Confirm the intended Azure tenant, subscription, and signed-in identity. Do not copy tenant or subscription IDs into repository files.
2. Confirm `az account show` and `azd env get-values` agree.
3. Use `azd` version `1.23.11` or newer.
4. Install Azure CLI, PowerShell, Python dependencies, and .NET SDK 10.
5. Register all Azure resource providers required by the Bicep modules.
6. Check model availability and quota in the requested Foundry region.
7. Check Search, Fabric, and App Service regional capacity and quota.
8. Confirm the Fabric user is licensed and the capacity reports `Active` through the Fabric API.
9. Confirm the deploying principal can create resources, assign required roles, and administer the Fabric workspace.
10. For private GitHub source, provide a process-scoped `GITHUB_TOKEN` with the minimum repository read permission. Never store it in `.env`, azd state, source control, command output, or documentation.

Fabric tenant settings must be enabled by a Fabric Administrator before ontology/data-agent deployment:

- Users can create Ontology (preview) items.
- Users can use Copilot and other features powered by Azure OpenAI.
- Data sent to Azure OpenAI can be processed outside the capacity geographic region when required.
- Data sent to Azure OpenAI can be stored outside the capacity geographic region when required.

Allow up to 15 minutes for tenant-setting propagation. A `403 FeatureNotAvailable` from the Ontology API means the feature is not enabled or available; changing Bicep or retrying the full installer will not fix it.

## Environment Setup

Use an isolated azd environment and resource group for each deployment. Prefer a short lowercase prefix that is safe in global Azure resource names.

```sh
azd env new <environment>
azd env set AZURE_SUBSCRIPTION_ID <subscription-id>
azd env set AZURE_RESOURCE_GROUP <resource-group>
azd env set AZURE_LOCATION <primary-region>
azd env set AZURE_AI_DEPLOYMENTS_LOCATION <foundry-region>
azd env set AZURE_SEARCH_SERVICE_LOCATION <search-region>
azd env set AZURE_APP_SERVICE_LOCATION <app-region>
```

Set model, SKU, capacity, authentication, and existing-resource values documented in `docs/DeploymentGuide.md`. Inspect values with:

```sh
azd env get-values
```

Do not paste that output into logs because it can contain sensitive configuration.

For a deployment sourced from a remote other than local `origin`, set:

```sh
export GITHUB_REPOSITORY=<owner>/<repository>
```

The Fabric notebook uploader honors this override. Without it, repository auto-detection reads local `origin`, which may point at the upstream repository rather than the intended private repository.

## Normal Deployment

Validate infrastructure before applying it:

```sh
azd provision --preview
```

Then run the complete deployment:

```sh
azd up
```

Deploy application packages again without reprovisioning when needed:

```sh
azd deploy web
azd deploy func
```

For direct postprovision recovery, use the active azd environment and repository override:

```sh
GITHUB_REPOSITORY=<owner>/<repository> \
PYTHONPATH=infra/scripts \
python3 infra/scripts/install_microsoft_iq_solution.py
```

Provide `GITHUB_TOKEN` only in the process environment when private repository access is required. Remove it immediately afterward.

## Private Repository Handling

A true GitHub fork can be blocked by organization SAML policy. A private standalone repository is an acceptable deployment source when forking is unavailable.

The local remotes can intentionally differ:

- `origin` may remain the upstream repository.
- A separate remote may point to the private deployment repository.

Always inspect `git remote -v` before pushing. Push explicit commits to the intended remote. Never rewrite or reset unrelated user work.

The notebook uploader injects a private-repository token into the uploaded notebook definition for the download call. After the repository has been downloaded and deployed:

1. Fetch the uploaded notebook definition through Fabric REST.
2. Remove the `GITHUB_TOKEN` assignment and `github_token=GITHUB_TOKEN` argument.
3. Update the notebook definition.
4. Fetch it again and verify `GITHUB_TOKEN` is absent.
5. Rotate the token if it might have been exposed.

## Partial Failure Rules

Azure and Fabric operations are idempotent but not atomic. A failed parent deployment can leave useful resources and data in place.

- Never delete the Fabric workspace or capacity just because the installer failed.
- Inventory live items before rerunning anything.
- Inspect the parent notebook job and child notebook jobs separately.
- A parent error such as `System_Cancelled_Session_Statements_Failed` does not identify the failing cell.
- `pipeline_main` creates lakehouse tables. If its job completed, do not rerun the data pipeline while repairing ontology or folder operations.
- Fabric Livy APIs expose session state and Spark application metadata but may not expose statement output. Use child job `failureReason` or the Fabric monitoring UI for cell-level diagnostics.
- Retry only the failed post-deployment stage whenever possible.

Expected Fabric ordering:

1. Lakehouse.
2. Notebooks, semantic models, and reports.
3. `pipeline_main` data/table creation.
4. Ontologies.
5. Data agents.
6. Folder moves and final organization.

A workspace with base items but no ontology/data agent usually means the post-deployment stage failed after data creation.

## Known Failure Patterns

### Search capacity unavailable

Move Search independently with `AZURE_SEARCH_SERVICE_LOCATION`. Do not move Foundry or every resource unless required.

### Fabric capacity quota or reserved name

Check the target region and SKU quota. A failed Fabric backend operation can temporarily reserve a capacity name; use a new globally valid name instead of repeatedly retrying the failed name.

### App Service quota unavailable

Move App Service independently with `AZURE_APP_SERVICE_LOCATION`.

### Function deployment succeeds but host fails

A successful package upload does not prove the Functions host can access storage. For managed-identity host storage, verify these built-in roles at storage-account scope:

- Storage Blob Data Owner.
- Storage Queue Data Contributor.
- Storage Account Contributor.

After role propagation, restart or redeploy the Function App so it obtains a fresh token. Verify host status and trigger registration.

### Fabric workspace is empty

Confirm both installer phases are active:

- `upload_installer_notebook(...)`
- `run_installer_notebook(...)`

Confirm direct execution loaded the active azd environment and the notebook points at the correct GitHub owner, repository, and branch.

### Fabric installer fails after several minutes

List workspace items first. Base deployment and `pipeline_main` may already have succeeded. Query the `pipeline_main` job independently, inspect OneLake, then isolate ontology/data-agent errors.

### Ontology returns `403 FeatureNotAvailable`

Verify the required Fabric tenant preview setting with a Fabric Administrator. An Azure subscription Owner, Contributor, or Global Reader cannot enable Fabric tenant settings unless also assigned the required Fabric administrator role.

### Foundry model request fails

Check model version, deployment type, regional availability, and quota. Keep sibling model settings consistent with Bicep parameter constraints.

## Required Validation

A deployment is complete only when every applicable check passes.

### Azure resource state

- Resource-group deployment state is `Succeeded`.
- Fabric ARM capacity is provisioned and Fabric API state is `Active`.
- Model deployments exist with the intended versions and capacity.
- Search, storage, messaging, monitoring, web, and Function resources exist in their configured regions.

### Fabric

- Workspace is assigned to the intended capacity.
- Item inventory includes the expected Lakehouse, SQL endpoint, notebooks, semantic models, reports, ontologies, and data agent.
- `pipeline_main` completed.
- OneLake `Files` contains source data.
- OneLake `Tables` contains Delta table directories and `_delta_log` files.
- Ontology definitions reference the target workspace and lakehouse IDs, not placeholders.
- Data-agent definitions reference the deployed ontology ID.

Use the Fabric REST item inventory and OneLake DFS API. Do not infer success from notebook upload alone.

### Azure AI Search

- Both expected indexes exist.
- Document counts are nonzero and match uploaded chunk totals.
- Both knowledge sources exist.
- Both knowledge bases exist.
- Query the Search data plane, not only installer logs.

### Foundry

- Expected agents exist in the intended project.
- Model deployments and agent tools resolve successfully.
- Existing-project reuse includes all required data-plane RBAC assignments.

### Applications

- Build the touched .NET project with SDK 10.
- Web endpoint returns HTTP 200.
- Function endpoint returns HTTP 200.
- Function host status is `Running`.
- Expected triggers are registered.
- Review Function logs for storage 403, startup, or indexing errors.

## Focused Validation Commands

Use commands that do not disclose environment values:

```sh
python3 -m py_compile <changed-python-files>
dotnet build src/web/Onboarding.Web.csproj --nologo
dotnet build src/func/Onboarding.FunctionApp.csproj --nologo
azd show
```

Use `az rest` or the repository API clients for Fabric and Search data-plane checks. Prefer resource-specific validation over rerunning `azd up` as a diagnostic.

## Bicep and Access Conventions

- Keep resource deployment under `infra/`.
- Use existing modules and naming conventions.
- Keep Search and storage public network access enabled when required by this solution.
- Add the `SecurityControl=Ignore` tag to Search and storage resources as required by repository policy.
- Use managed identities and built-in roles where possible.
- Scope role assignments to the narrowest resource that satisfies runtime behavior.
- Do not remove a stale role assignment while troubleshooting unless the user explicitly requests cleanup; first add and validate the required role.
- Use unique deterministic role-assignment names.

## Commit Discipline

Before committing:

1. Run file diagnostics.
2. Run the narrowest executable validation.
3. Review `git diff -- <intended-files>`.
4. Review `git status --short`.
5. Stage only intended files.
6. Push to the explicitly requested remote and branch.
7. Verify the remote ref with `git ls-remote`.

Do not include generated files, local azd state, `.env` files, tokens, notebook credentials, build output, `.vscode/`, or unrelated dirty files.

## Operator Handoff

Report deployment status by subsystem instead of saying only "deployed":

- Infrastructure provisioned.
- Application packages deployed.
- Search populated and counts verified.
- Foundry models and agents verified.
- Fabric base items deployed.
- Fabric data/tables verified.
- Ontologies verified.
- Data agent verified.
- Endpoints healthy.
- Remaining external prerequisites or permission blockers.

Include the exact failing API status and the minimum administrator action for external blockers. Never claim end-to-end completion while a required subsystem is absent.
