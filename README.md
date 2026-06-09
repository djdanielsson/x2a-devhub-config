# x2a-devhub-config

Backstage catalog and template configuration for the X2Ansible Developer Hub.

## Repository Structure

```
├── catalog-info.yaml                    # Component entity (catalog listing)
├── templates/
│   └── x2a-migration-template.yaml      # Scaffolder template (the wizard UI)
└── tekton/
    ├── kustomization.yaml               # Kustomize entrypoint for Tekton resources
    ├── task.yaml                         # Tekton Task: runs x2a-convertor phases
    └── pipeline.yaml                     # Tekton Pipeline: clone -> init -> analyze -> migrate -> push
```

## How It Works

The Developer Hub (RHDH) loads two catalog locations from this repo:

| File | Backstage Kind | What It Does |
|------|---------------|--------------|
| `catalog-info.yaml` | `Component` | Registers x2a-convertor in the software catalog |
| `templates/x2a-migration-template.yaml` | `Template` | Provides the migration wizard under "Create" |

When a user fills out the template wizard and clicks "Create", the scaffolder action
`kubernetes:create-resource` creates a Tekton `PipelineRun` in the `x2a-devhub` namespace.
The pipeline runs four sequential tasks:

1. **clone-source** -- Clones the user's Chef/Puppet/Salt repository
2. **init** -- Runs `x2a-convertor init` to generate migration metadata
3. **analyze** -- Runs `x2a-convertor analyze` to produce per-module migration plans
4. **migrate** -- Runs `x2a-convertor migrate` for each module plan
5. **push-results** -- Commits and pushes the Ansible output to the target repository

## Deployment

### 1. Deploy the Tekton resources

Tekton resources are deployed automatically via ArgoCD. The `x2a-devhub-tekton` Application
in `ocp_lab/apps/argocd/applications/` syncs the `tekton/` directory to the `x2a-devhub` namespace.

### 2. Point the Dev Hub at this repo

The RHDH Helm values in `ocp_lab/apps/x2a-devhub/values.yaml` are already configured to
point at this repo (`djdanielsson/x2a-devhub-config`).

### 3. Ensure prerequisites

- **OpenShift Pipelines** operator is installed (provides Tekton + `git-clone`/`git-cli` ClusterTasks)
- **`kubernetes:create-resource`** scaffolder action is enabled in RHDH (comes from the `@backstage/plugin-scaffolder-backend-module-kubernetes` plugin)
- **Ollama** is accessible at `http://ollama.ollama.svc.cluster.local:11434/v1` from the `x2a-devhub` namespace
- The **x2a-convertor container image** (`quay.io/x2ansible/x2a-convertor:latest`) is available

## Customization

- To change available AI models, edit the `enum`/`enumNames` in `templates/x2a-migration-template.yaml`
- To change the Ollama endpoint, edit the `OPENAI_API_BASE` env var in `tekton/task.yaml`
- To add more source technologies, add values to the `sourceTechnology` enum in the template
