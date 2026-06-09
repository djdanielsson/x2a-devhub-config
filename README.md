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
    ├── pipeline.yaml                     # Tekton Pipeline: clone -> init -> analyze -> migrate -> push
    └── triggers.yaml                     # EventListener, TriggerTemplate, TriggerBinding, RBAC
```

## How It Works

The Developer Hub (RHDH) loads two catalog locations from this repo:

| File | Backstage Kind | What It Does |
|------|---------------|--------------|
| `catalog-info.yaml` | `Component` | Registers x2a-convertor in the software catalog |
| `templates/x2a-migration-template.yaml` | `Template` | Provides the migration wizard under "Create" |

When a user fills out the template wizard and clicks "Create":

1. The `http:backstage:request` scaffolder action POSTs the form data to a Backstage proxy endpoint
2. The proxy forwards the request to the Tekton EventListener (`el-x2a-migration`) in the cluster
3. The EventListener creates a PipelineRun via its TriggerTemplate
4. The pipeline runs: **clone-source** -> **init** -> **analyze** -> **migrate** -> **push-results**

## Deployment

### 1. Deploy the Tekton resources

Tekton resources are deployed automatically via ArgoCD. The `x2a-devhub-tekton` Application
in `ocp_lab/apps/argocd/applications/` syncs the `tekton/` directory to the `x2a-devhub` namespace.

### 2. Point the Dev Hub at this repo

The RHDH Helm values in `ocp_lab/apps/x2a-devhub/values.yaml` are already configured to
point at this repo (`djdanielsson/x2a-devhub-config`).

### 3. Ensure prerequisites

- **OpenShift Pipelines** operator is installed (provides Tekton + ClusterTasks + Triggers)
- **`http:backstage:request`** plugin is enabled in RHDH dynamic plugins config
- **Backstage proxy** is configured with `/x2a-trigger` pointing to the EventListener service
- **Ollama** is accessible at `http://ollama.ollama.svc.cluster.local:11434/v1`
- The **x2a-convertor container image** (`quay.io/x2ansible/x2a-convertor:latest`) is available

## Customization

- To change available AI models, edit the `enum`/`enumNames` in `templates/x2a-migration-template.yaml`
- To change the Ollama endpoint, edit the `OPENAI_API_BASE` env var in `tekton/task.yaml`
- To add more source technologies, add values to the `sourceTechnology` enum in the template
