<div align="center">

# Oscar-infra

**End-to-end GitOps pipeline running on a Raspberry Pi 5**

Push code, get a built arm64 image, and watch Flux roll it out to a self-hosted Kubernetes cluster. No manual `kubectl apply`.

![Build](https://img.shields.io/github/actions/workflow/status/Lordsofzzzz/Oscar-infra/ci.yml?label=build&logo=githubactions&logoColor=white)
![Flux](https://img.shields.io/badge/GitOps-Flux-5468FF?logo=flux&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![GHCR](https://img.shields.io/badge/Registry-GHCR-181717?logo=github&logoColor=white)
![Cloudflare Tunnel](https://img.shields.io/badge/Cloudflare-Tunnel-F38020?logo=cloudflare&logoColor=white)
![Raspberry Pi 5](https://img.shields.io/badge/Raspberry%20Pi-5%20(arm64)-C51A4A?logo=raspberrypi&logoColor=white)

</div>

---

## Overview

This repository is the **infrastructure and deployment layer** for [Oscar](https://github.com/c0dem0de/Oscar), a two-tier web application (backend API + web frontend). It contains:

- Kubernetes manifests for the app, managed declaratively with **Flux CD**
- A **GitHub Actions** pipeline that builds multi-service container images and updates the manifests automatically
- Everything needed to rebuild the cluster state from Git alone

Both Flux and the application run on a single **Raspberry Pi 5**, so the whole delivery loop is self-hosted, low-cost, and reproducible. The app is exposed to the internet through a **Cloudflare Tunnel**, so the Pi needs no open inbound ports and no port forwarding on the home router.

## Architecture

```mermaid
flowchart LR
    dev([Developer])

    subgraph GH["GitHub"]
        app["c0dem0de/Oscar<br/>app source code"]
        infra["Lordsofzzzz/Oscar-infra<br/>manifests + CI"]
        ci["GitHub Actions<br/>build workflow"]
        ghcr[("GHCR<br/>container images")]
    end

    subgraph PI["Raspberry Pi 5 (arm64)"]
        cfd["cloudflared<br/>runs on the Pi host"]
        subgraph K8S["Kubernetes cluster"]
            flux["Flux CD<br/>source + kustomize controllers"]
            subgraph NS["namespace: oscar"]
                fe["oscar-frontend<br/>NodePort 30173"]
                be["oscar-backend<br/>ClusterIP 5677"]
            end
        end
    end

    cf["Cloudflare edge<br/>public hostname + TLS"]
    user([User browser])

    dev -->|git push| app
    app -->|"repository_dispatch: app-build"| ci
    ci -->|"build and push arm64 images"| ghcr
    ci -->|"commit new image tags"| infra
    flux -->|"poll every 1m"| infra
    flux -->|apply| NS
    ghcr -->|image pull| NS
    user -->|HTTPS| cf
    cf <-->|"outbound-only tunnel"| cfd
    cfd -->|"localhost:30173"| fe
    fe -.->|in-cluster API calls| be
```

## CI/CD flow

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant App as Oscar (app repo)
    participant CI as GitHub Actions
    participant Reg as GHCR
    participant Infra as Oscar-infra (this repo)
    participant Flux as Flux on Pi 5
    participant K8s as Kubernetes

    Dev->>App: push code
    App->>CI: repository_dispatch (app-build)
    CI->>App: checkout source
    CI->>CI: build backend + frontend (linux/arm64)
    CI->>Reg: push images tagged main-sha7
    CI->>Infra: bump image tags, commit "deploy: main-sha7"
    Flux->>Infra: poll for changes (1m)
    Infra-->>Flux: new commit
    Flux->>K8s: apply updated manifests
    K8s->>Reg: pull new images
    K8s-->>Flux: rollout complete
```

Git is the single source of truth. To change what runs in the cluster, change a manifest here and let Flux apply it.

## Tech stack

| Layer | Tool |
| --- | --- |
| Hardware | Raspberry Pi 5 (arm64) |
| Orchestration | Kubernetes |
| GitOps controller | Flux CD |
| CI | GitHub Actions (Buildx + QEMU) |
| Container registry | GitHub Container Registry (GHCR) |
| Public access | Cloudflare Tunnel (`cloudflared`) |
| Config | Plain Kubernetes YAML + Kustomize (via Flux) |

## Repository layout

```mermaid
flowchart TD
    root["Oscar-infra"]
    gh[".github/workflows/ci.yml<br/>build and deploy pipeline"]
    clusters["clusters/my-cluster"]
    fs["flux-system/<br/>Flux sync manifests (generated)"]
    osc["oscar/"]
    ns["namespace.yml"]
    be["backend.yml<br/>Deployment + Service"]
    fe["frontend.yml<br/>Deployment + Service"]

    root --> gh
    root --> clusters
    clusters --> fs
    clusters --> osc
    osc --> ns
    osc --> be
    osc --> fe
```

## Workloads

| Component | Image | Port | Exposure |
| --- | --- | --- | --- |
| `oscar-backend` | `ghcr.io/lordsofzzzz/oscar-backend` | 5677 | ClusterIP (internal only) |
| `oscar-frontend` | `ghcr.io/lordsofzzzz/oscar-frontend` | 80 | NodePort `30173` |

Both run in the `oscar` namespace with TCP readiness and liveness probes and resource requests and limits sized for a Pi.

| Component | CPU request | Memory request | Memory limit |
| --- | --- | --- | --- |
| backend | 50m | 128Mi | 512Mi |
| frontend | 25m | 64Mi | 256Mi |

**Access**
- Public: through the Cloudflare Tunnel hostname (HTTPS, terminated at Cloudflare's edge).
- Local network: `http://<pi-ip>:30173`.

## Public access with Cloudflare Tunnel

The Pi sits behind a home network, so instead of port forwarding, `cloudflared`, which runs directly on the Pi host (not as a pod), opens an **outbound** connection to Cloudflare. Cloudflare routes requests for the public hostname through that tunnel to the frontend NodePort on the Pi.

```mermaid
flowchart LR
    u([User]) -->|"https://your-domain"| cf["Cloudflare edge<br/>TLS + DDoS protection"]
    cf <-->|"encrypted outbound tunnel"| cfd["cloudflared<br/>runs directly on the Pi host"]
    cfd -->|"localhost:30173"| fe["oscar-frontend<br/>NodePort"]
    fe -.-> be["oscar-backend<br/>ClusterIP"]
```

Why a tunnel:
- No inbound ports opened on the router, so the Pi's IP is never exposed.
- Works behind CGNAT and dynamic IPs.
- TLS certificates and DDoS protection come from Cloudflare at no extra setup.
- The backend stays internal (`ClusterIP`) and is never published directly.

## Pipeline details

Workflow: [`.github/workflows/ci.yml`](.github/workflows/ci.yml)

**Triggers**
- `repository_dispatch` of type `app-build`, sent by the app repo
- `workflow_dispatch` for manual runs from the Actions tab

**What it does**
1. Checks out the private app repo using a read-only token.
2. Logs in to GHCR with the built-in `GITHUB_TOKEN`.
3. Derives the image tag `main-<7-char app commit SHA>`, so every image maps back to an exact source commit.
4. Builds and pushes backend and frontend images for `linux/arm64`.
5. Rewrites the `image:` lines in the manifests and pushes a `deploy: <tag>` commit as `ci-bot`.

**Required secret**

| Secret | Purpose |
| --- | --- |
| `APP_REPO_READ_TOKEN` | Read access to the private app repo |

## Flux configuration

Defined in `clusters/my-cluster/flux-system/gotk-sync.yaml`:

| Setting | Value |
| --- | --- |
| Git source | `ssh://git@github.com/Lordsofzzzz/Oscar-infra` (branch `main`) |
| Source poll interval | 1 minute |
| Kustomization path | `./clusters/my-cluster` |
| Reconcile interval | 10 minutes |
| Pruning | Enabled. Resources removed from Git are removed from the cluster |

## Getting started

**Prerequisites**
- Raspberry Pi 5 (or any arm64 host) running Kubernetes
- `kubectl` and the [`flux` CLI](https://fluxcd.io/flux/installation/)
- A GitHub token or deploy key Flux can use

**Bootstrap Flux**

```bash
flux bootstrap github \
  --owner=Lordsofzzzz \
  --repository=Oscar-infra \
  --branch=main \
  --path=clusters/my-cluster
```

Flux installs itself, then reconciles everything under `clusters/my-cluster`, including the `oscar` namespace and both workloads.

## Operations

```bash
# Overall Flux health
flux get kustomizations
flux get sources git

# App status
kubectl -n oscar get pods,svc
kubectl -n oscar logs deploy/oscar-backend

# Force an immediate sync
flux reconcile source git flux-system
flux reconcile kustomization flux-system
```

| Task | How |
| --- | --- |
| Deploy a new version | Push to the app repo, or run the `build` workflow manually |
| Roll back | `git revert` the `deploy:` commit on `main`. Flux applies the previous tag |
| Scale or tune resources | Edit the manifest, commit, and let Flux apply it |

## Design decisions

- **GitOps over push-based deploys.** The cluster pulls its desired state from Git. No cluster credentials are stored in CI, and every change is auditable and revertable.
- **Separate app and infra repos.** Application code and deployment config change at different rates and need different access controls.
- **Immutable, traceable tags.** Images are tagged with the source commit SHA, never `latest`, so a running pod always maps to an exact commit.
- **Native arm64 images.** Built with Buildx and QEMU for the Pi 5, avoiding emulation at runtime.
- **Zero inbound exposure.** Cloudflare Tunnel uses outbound connections only, which removes router port forwarding and hides the Pi's real IP.
- **Right-sized resources.** Requests and memory limits keep both services predictable on a small single-node cluster.

## Notes

- CI rewrites image tags with `sed`. Keep the `image: ghcr.io/lordsofzzzz/oscar-<service>:<tag>` line format intact.
- The backend port `5677` appears several times in `backend.yml`. Change all occurrences together.
- Do not hand-edit generated files in `flux-system/`.

## Roadmap

- [ ] Ingress controller behind the tunnel instead of a raw NodePort
- [ ] Move `cloudflared` into the cluster as a Flux-managed Deployment
- [ ] Encrypted secrets with SOPS or Sealed Secrets
- [ ] Monitoring with Prometheus and Grafana
- [ ] Image update automation and rollout notifications
- [ ] Staging environment

## License

TODO: add a license.
