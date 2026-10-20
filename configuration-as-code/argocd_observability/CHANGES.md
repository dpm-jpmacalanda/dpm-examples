# Changes in this fork

This folder is a modified copy of `configuration-as-code/argocd_observability` from
[Dynatrace/community-examples](https://github.com/Dynatrace/community-examples)
(upstream commit `3e1a6ee`), licensed under the Apache License 2.0 (see `LICENSE.txt` at the repository root).

Modified by DPM International Inc., October 2026.

## Modified file

`argocd_observability/openpipeline/events.sdlc.argocd.pipelines.application.json`
(pipeline `pipeline_argocd_application_v2`, display name "ArgoCD Application")

Processor order:

1. `processor_add_argocd_deployment_properties` (upstream, name and stage lines changed)
2. `processor_enrich_argocd_event` (new; must run before Clean up, which removes the raw `app` field)
3. `processor_clean_up` (upstream, unchanged)

### 1. Name and stage labels (processor 1)

| Field | Before | Now |
|---|---|---|
| `argocd.app.name` | label `app` | label `app`, else label `name` |
| `argocd.app.stage` (and `cicd.deployment.release_stage`) | label `stage` | label `stage`, else label `environment` |

### 2. New and filled fields (processor 2)

| Field | Source in the Argo CD Application | Example |
|---|---|---|
| `argocd.app.images` | `status.summary.images` | all images of the app |
| `argocd.app.image` | first image whose name does not contain `dynatrace` (skips the OneAgent injection image) | `<registry>/team/app:dev-6cc3ce6` |
| `argocd.app.image.tag` | tag (after the last `:`) or digest (after `@`) of `argocd.app.image` | `dev-6cc3ce6` |
| `argocd.app.version` | **the image tag**; the `version` label only if the app has no image | `dev-6cc3ce6` |
| `argocd.app.owner` | label `owner`, else `spec.project` | `team-project` |
| `vcs.ref.base.name` | unchanged, else `spec.source.targetRevision` (fills "started" events) | `main` |
| `vcs.repository.url.full` | unchanged, else `spec.source.repoURL` (fills "started" events) | GitOps repository URL |
| `argocd.sync.initiated_by` | `status.operationState.operation.initiatedBy.username`, or `automated` | user name / e-mail, or `automated` |
| `argocd.app.deployment_count` | `id` of the last `status.history` entry | `98` |
| `vcs.ref.previous.revision` | revision of the second-to-last `status.history` entry | Git commit |
| `cicd.deployment.ingress.url` | first `status.summary.externalURLs` entry | app URL |
| `argocd.app.source.path` | `spec.source.path` | chart path in the GitOps repo |
| `argocd.app.source.type` | `status.sourceType` | `Helm` |
| `argocd.app.helm.release` | `spec.source.helm.releaseName` | release name |
| `argocd.app.helm.value_files` | `spec.source.helm.valueFiles` | list of values files |
| `argocd.app.parent` | annotation `argocd.argoproj.io/tracking-id`, text before the first `:` | parent (app-of-apps) name |
| `argocd.sync.auto` | `spec.syncPolicy.automated` is set | `true` |
| `argocd.sync.self_heal`, `argocd.sync.prune` | `spec.syncPolicy.automated.selfHeal` / `.prune` | `true` |
| `argocd.sync.retry_limit` | `spec.syncPolicy.retry.limit` | `3` |
| `argocd.sync.resources` | `status.operationState.syncResult.resources` as `Kind/name` | `Deployment/my-app`, … |
| `argocd.app.health.since` | `status.health.lastTransitionTime` | timestamp |
| `argocd.app.created` | `metadata.creationTimestamp` | timestamp |

Notes:

- `argocd.sync.initiated_by` stores the Argo CD user name, which is often an e-mail address. Remove that line if personal data must not be stored.
- Apps with several application containers: `argocd.app.image` and the tag describe the first one; the full list stays in `argocd.app.images`.
- Multi-source Applications (`spec.sources`) are not handled; the source fields stay empty for them.

### 3. Dashboard "ArgoCD Application Lifecycle" extended

`argocd_observability/document-dashboard/argocd_app_lifecycle_observability.json`:

- "Last Application Sync Operation": adds Version and DeployedBy
- "Finished deployments | last 20": adds Version, DeployedBy, DeploymentNo, PreviousRev, Image, Stage, Parent, ChartPath, AppURL and Resources
- "Last Failed Operation States": adds version and initiator, and passes them to the CoPilot prompt context
- New section "Versions and Releases": current version per application; all applications with health, sync and version (apps without events show "not subscribed"); apps waiting for promotion (Suspended); deployments by initiator; version history
- About text points to the current upstream location and this fork

The "ArgoCD Platform Observability" dashboard is unchanged.

### 4. Documentation

- `docs/Dynatrace_ArgoCD_Integration_Runbook_v2.1.html`: generic implementation runbook (no customer data), linked from the README
- `README.md`: "DPM additions" section at the top; the original Dynatrace README follows unchanged

## Unchanged

Pipeline ID, ingest source (`/platform/ingest/custom/events.sdlc/argocd`), the Platform Observability dashboard, manifest.
Redeploying this project over an existing upstream deployment updates the same pipeline in place;
the Argo CD side needs no change. Processors added manually in the UI to this pipeline are replaced.

## Example query

```
fetch events, from: now() - 7d
| filter event.kind == "SDLC_EVENT" and event.provider == "argocd" and event.status == "finished"
| fields timestamp, cicd.deployment.name, argocd.app.version, argocd.app.image,
         argocd.sync.initiated_by, argocd.app.deployment_count, vcs.ref.base.revision, task.outcome
| sort timestamp desc
```
