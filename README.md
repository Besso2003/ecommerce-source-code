**Note:** This project is a fork of `opentelemetry-demo`. Thanks to the team and contributors for opensourcing this wonderful demo project. Definitely one of the best on internet.

## Documentation

For detailed documentation, see [Demo Documentation][docs]. If you're curious
about a specific feature, the [docs landing page][docs] can point you in the
right direction.

## Demos featuring the Astronomy Shop

We welcome any vendor to fork the project to demonstrate their services and
adding a link below. The community is committed to maintaining the project and
keeping it up to date for you.

|                           |                |                                  |
|---------------------------|----------------|----------------------------------|
| [AlibabaCloud LogService] | [Elastic]      | [OpenSearch]                     |
| [AppDynamics]             | [Google Cloud] | [Sentry]                         |
| [Aspecto]                 | [Grafana Labs] | [ServiceNow Cloud Observability] |
| [Axiom]                   | [Guance]       | [Splunk]                         |
| [Axoflow]                 | [Honeycomb.io] | [Sumo Logic]                     |
| [Azure Data Explorer]     | [Instana]      | [TelemetryHub]                   |
| [Coralogix]               | [Kloudfuse]    | [Teletrace]                      |
| [Dash0]                   | [Liatrio]      | [Tracetest]                      |
| [Datadog]                 | [Logz.io]      | [Uptrace]                        |
| [Dynatrace]               | [New Relic]    |                                  |

---

# CI/CD Pipeline

This repo contains the application source code for the ecommerce demo's microservices. Pushing changes to `main` automatically builds and deploys affected services to the `dev` environment, defined in the separate [`ecommerce-aws-eks-devops`](https://github.com/Besso2003/ecommerce-aws-eks-devops) repo.

## What happens on every push to `main`

```
1. detect-changes   — figures out which of the 15 tracked services
                       actually had files change in this push
2. build-and-push    — for each changed service, in parallel:
                          builds its Docker image, tags it with the
                          short git SHA, pushes both that tag and
                          :latest to its ECR repository in ecommerce-dev
3. update-manifests   — clones ecommerce-aws-eks-devops, updates the
                          image tag for every service that was just
                          built, and pushes ONE combined commit back
                          to that repo
```

ArgoCD (running in the separate `platform` hub cluster) watches `ecommerce-aws-eks-devops` and picks up that commit automatically — no further action is needed for a change to reach the `dev` cluster.

Only `dev` is touched by this pipeline. Promoting a tested image to `prod` is a separate, deliberate step, not automated by this workflow.

## Why services are built from the repo root, not their own folder

Every service's `Dockerfile` (including ones that look self-contained at first glance) references shared files outside its own folder — most commonly `pb/demo.proto`, the protobuf contract shared across services. Because of this, every build in this pipeline uses the **repo root** as its Docker build context, even though the `Dockerfile` itself lives inside `src/<service>/`. The `-f` flag points at the Dockerfile's actual location; the context (what files Docker can see) is always `.`.

## Per-service quirks the pipeline accounts for

```
cart            — Dockerfile lives at src/cart/src/Dockerfile,
                   not src/cart/Dockerfile (a .NET project
                   convention from the upstream demo project)

ad              — folder is named "ad", but its ECR repository
                   and Kubernetes deployment are named "ad-service"

recommendation  — folder is named "recommendation", but its ECR
                   repository and Kubernetes deployment are named
                   "recommendation-service"
```

These three are handled explicitly in the workflow's case statements. Every other service's folder name, ECR repository name, and Dockerfile path all match the simple pattern `src/<service>/Dockerfile` → `ecommerce-dev/<service>`.

## Services NOT covered by this pipeline

```
flagd, flagd-ui     — third-party images (open-feature project),
                       not built from this repo's code

kafka, postgres,
valkey               — third-party / infrastructure images, not
                       application code

prometheus, grafana,
jaeger, opensearch,
otel-collector         — observability stack, not part of the
                          23 services currently deployed via
                          k8s/overlays in the infra repo

react-native-app        — has no corresponding Kubernetes
                            deployment currently

product-reviews, llm    — referenced in the infra repo's k8s
                            manifests at one point, but have no
                            corresponding folder in this repo's
                            src/ directory. product-reviews was
                            removed from the dev overlay (see
                            "Known issues" below); llm has not
                            been investigated yet.
```

## How authentication works (no stored AWS keys)

The workflow authenticates to AWS using **OIDC federation**: GitHub issues a short-lived, cryptographically signed token identifying the exact repo and branch the workflow is running from. AWS validates that token against an IAM role whose trust policy only accepts tokens from `repo:Besso2003/ecommerce-source-code:ref:refs/heads/main`. No AWS access key or secret is stored anywhere in this repo or its GitHub secrets.

The IAM role itself is provisioned in the infra repo's Terraform (`Terraform/bootstrap/`), and is scoped narrowly — it can push images to `ecommerce-dev/*` ECR repositories only. It cannot touch `ecommerce-prod/*`, and has no permissions outside ECR at all.

## How the cross-repo commit works

Pushing a commit into `ecommerce-aws-eks-devops` from a workflow running in this repo requires write access to that other repo. This is handled by a dedicated **GitHub App** (`ecommerce-ci-bot`), installed on both repos with `Contents: Read and write` permission only. The workflow exchanges the App's credentials for a short-lived (~1 hour) installation token at runtime, used only for that one commit. The App's private key is stored as a GitHub Actions secret (`CI_BOT_PRIVATE_KEY`) and is never written to disk outside the workflow's own memory.

## Known issues

- **`product-reviews`** was deployed in the dev overlay but its server code expected a `ProductReviewService` gRPC contract that was never added to the shared `pb/demo.proto` file. This caused a permanent `CrashLoopBackOff`. It has been removed from `k8s/overlays/dev/kustomization.yaml` until the protobuf definition is written and the service is properly finished.
- **`llm`** has no corresponding folder in this repo's `src/` directory despite being referenced in the infra repo's deployments. Not yet investigated.

## Adding a new service to the pipeline

1. Add the service's folder name to the `filters:` list in the `detect-changes` job of `.github/workflows/build-and-push-dev.yml`
2. If its Dockerfile path or ECR name doesn't follow the standard `src/<service>/Dockerfile` → `ecommerce-dev/<service>` pattern, add a case for it in both the `build-and-push` job's case statement and the `update-manifests` job's `ecr_name_for()` function
3. Make sure a matching `k8s/overlays/dev/patches/<service>-patch.yaml` exists in the infra repo — the pipeline will insert a new `image:` line automatically the first time it builds that service, but only if the patch file itself already exists