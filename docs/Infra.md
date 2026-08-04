# Infrastructure & Deployment Flow

Our standard deployment process for other apps is
[here](https://docs.hotosm.org/devops/deployment-process)

fAIr differs slightly, because we have:
- Versioning of both software, as well as AI models.
- A dedicated dev instance EC2 for easier development with all components.

Currently model development happens in the `fAIr-models` repo, but this
might eventually move to the `fAIr` monorepo.

The model flow works like this, explicitly separating **infrastructure state** from **model catalog state**:

* **Smart CI Builds:** A GitHub Actions workflow uses a dynamic `git diff` strategy to detect which model directories under `./models` have changed (falling back to building all models if shared core files change). It builds the Docker images and tags them with the git SHA.
* **Infrastructure Deployment (GitOps):** The CI pipeline pushes the newly built image tags to the `values.yaml` file in the `k8s-infra` repository. ArgoCD detects this and automatically spins up/updates the Knative inference endpoints so the models are physically running on the cluster.
* **Model Catalog (Source of Truth):** While ArgoCD manages the physical infrastructure, the **STAC database remains the absolute source of truth** for our model catalog. Automated STAC registration via the Helm chart is explicitly disabled (`stac.register: false`) to protect the database from being polluted by CI/CD runs.
* **Django Administration:** Instead, we use the Django admin to give a deployed SHA a version (`vX.Y.Z-rc.N`, then `vX.Y.Z`) and register it in the STAC, pinned to the image digest (`rc` release candidates are used for staging, before full production tagging).
* A `BaseModel` table holds the model name and its status. The version details live entirely in the STAC, however.
  
## Step 1: Development

> [!NOTE]
> The Environment
> - Single EC2, lightweight k3s cluster.
> - Manually updated / synced with dev.
> - Model registration in STAC etc is all manual.

1. Users work on models in development, versioned as `-dev`
   with a specific SHA tag too.
2. Development model image (deps + code) is pushed to GHCR.
3. On the dev EC2 they run a script to update the **dev** STAC
   and knative records.
4. Any changes to the frontend / API are manually synced to
   the dev EC2 instance.
5. The dev model can be tested on the dev instance, using the
   dev STAC, ZenML, knative services.

## Step 2: Staging

> [!NOTE]
> The Environment
> - Runs all the same components as production, but
>   start up via PR from `staging` --> `main`.
> - The components run inside the `fair-staging`
>   namespace of the Kubernetes cluster, under
>   domain `https://stage.ai.hotosm.org`.
> - Does not run it's own `knative` controller,
>   instead using the cluster-wide instance.

1. When we want to stabilise and push out a **new model**, or **updates to the
   API / website**, we use the staging setup.
2. First a PR must be raised on the fAIr repo from `staging` --> `main`.
   This will set up `https://stage.ai.hotosm.org` with ZenML / STAC /
   Knative registration.
3. On boot the **staging** STAC is seeded (read-only) from the current
   **production** STAC, so it mirrors live.
4. CI has already built the model image, tagged by SHA. In the Django admin,
   give that SHA a candidate version (`vX.Y.Z-rc.N`), register it in the
   staging STAC, and test it.
5. Once it looks good, register the model in production (Step 3) before merging
   the PR to `main`. Merging shuts the staging env down.

## Step 3: Production

> [!NOTE]
> The Environment
> - Runs through tagged releases on Github, where ArgoCD
>   picks up the latest helm chart tag and deploys.

1. A new tagged version is made from the latest `main` code.
2. This triggers a redeploy of the fAIr website / API.
3. In the production Django admin, give the tested SHA a release version
   (`vX.Y.Z`), register its STAC item pinned to the digest, and make it live.
   The image is already in GHCR, so it is available straight away.
