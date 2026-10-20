# Changes in this fork

This folder is a modified copy of `configuration-as-code/argocd_observability` from
[Dynatrace/community-examples](https://github.com/Dynatrace/community-examples)
(upstream commit `3e1a6ee`), licensed under the Apache License 2.0 (see `LICENSE.txt` at the repository root).

Modified by DPM International Inc., October 2026.

## Modified file

`argocd_observability/openpipeline/events.sdlc.argocd.pipelines.application.json`
(pipeline `pipeline_argocd_application_v2`, display name "ArgoCD Application")

### 1. Container image and tag (new processor)

New DQL processor `processor_extract_argocd_images`, placed **between**
`processor_add_argocd_deployment_properties` and `processor_clean_up`
(it must run before Clean up, which removes the raw `app` field).

It adds these fields to each Argo CD SDLC event:

| Field | Source | Example |
|---|---|---|
| `argocd.app.images` | `app.status.summary.images` (all images) | `["<registry>/team/app:dev-6cc3ce6", "public.ecr.aws/dynatrace/dynatrace-operator:v1.10.2"]` |
| `argocd.app.image` | First image whose name does not contain `dynatrace` (skips the image added by Dynatrace OneAgent injection) | `<registry>/team/app:dev-6cc3ce6` |
| `argocd.app.image.tag` | Tag of `argocd.app.image` (text after the last `:`), or the digest after `@` | `dev-6cc3ce6` |

Apps with several application containers: `argocd.app.image` and `argocd.app.image.tag` describe the
first one only; the full list stays in `argocd.app.images`.

### 2. Application name and stage labels

The upstream design (`deployment_event.md`) maps `argocd.app.name` to `app.metadata.labels.name`,
but the script read `labels.app`. The script now accepts both common conventions:

- `argocd.app.name  = coalesce(labels.app,   labels.name)`
- `argocd.app.stage = coalesce(labels.stage, labels.environment)`

Applications that use the upstream labels behave exactly as before.

## Unchanged

Pipeline ID, ingest source (`/platform/ingest/custom/events.sdlc/argocd`), dashboards, manifest.
Redeploying this project over an existing upstream deployment updates the same pipeline in place;
the Argo CD side needs no change. Any processor added manually in the UI to this pipeline is replaced.

## Example query

```
fetch events, from: now() - 7d
| filter event.kind == "SDLC_EVENT" and event.provider == "argocd" and event.status == "finished"
| fields timestamp, cicd.deployment.name, argocd.app.image.tag, argocd.app.image, vcs.ref.base.revision, task.outcome
| sort timestamp desc
```
