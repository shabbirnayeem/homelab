# homelab

GitOps config for a single-node [k3s](https://k3s.io) cluster on a Raspberry Pi 5, managed by
[Flux](https://fluxcd.io) (v2.10). The app code it deploys lives in separate repos (e.g.
[`aws-resume`](https://github.com/shabbirnayeem/aws-resume)) — this repo only holds what runs on the cluster.

## Live traffic and CI

```
                         ┌─────────────────────── k3s on Raspberry Pi 5 ────────────────────────────┐
                         │                                                                           │
Browser ─► Cloudflare ◄──┼─ outbound tunnel ── cloudflared ──► portfolio-web Svc ──► portfolio-web pod │
                         │    (cloudflare ns)                   (portfolio ns)      (Flask, gunicorn) │
                         │                                                                           │
GitHub Actions ◄───────────── listener ── controller ──creates──► runner pod                          │
  (job queued)           │    (arc-systems ns)                    (arc-runners ns, ephemeral)         │
                         └───────────────────────────────────────────────────────────────────────────┘
```

Everything in this diagram is Flux-managed. `minRunners: 0` on the ARC scale set, so no runner pod exists until a
job is actually queued.

## Deploy pipeline (crosses two repos)

```
aws-resume (git)                                          homelab (git)
    │
    │ push to main (app/** or Dockerfile changed)
    ▼
GitHub Actions: test ──► build-and-push (multi-arch) ──► Docker Hub
                                                               │  tag: sha-<7-char-hash>-<run#>
                                                               ▼
                                            ImageRepository  (scans Docker Hub, flux-system ns)
                                                               │
                                                               ▼
                                            ImagePolicy      (numerical policy, highest run# wins)
                                                               │
                                                               ▼
                                            ImageUpdateAutomation ──commits new tag──► homelab main
                                                                                           │
                                                                                           ▼
                                                                    Kustomization "portfolio-app" reconciles
                                                                                           │
                                                                                           ▼
                                                                    portfolio-web Deployment rolling update
```

The CI job that builds and pushes the image holds **no Kubernetes credentials at all**. Flux, authenticated to
GitHub with its own deploy key, does the entire cluster-side half of a deploy.

## Repo layout

| Path | What it is |
|---|---|
| `clusters/pi/flux-system/` | Flux's own bootstrap manifests. Auto-managed by `flux bootstrap`/`flux` itself — never hand-edited. |
| `clusters/pi/apps/portfolio/` | Flux `Kustomization` + image-automation CRs (`ImageRepository`/`ImagePolicy`/`ImageUpdateAutomation`) for the portfolio app |
| `clusters/pi/infrastructure/` | Flux `Kustomization` CRs for cluster-wide infrastructure (ARC, cloudflared) |
| `apps/portfolio/` | The app's actual manifests: `Deployment`, `Service` |
| `infrastructure/arc/controllers/` | `HelmRelease` for the ARC controller |
| `infrastructure/arc/configs/` | `HelmRelease` for the `pi-k3s` runner scale set |
| `infrastructure/cloudflare/` | `Namespace` + `Deployment` for the Cloudflare Tunnel connector |

## Namespaces

| Namespace | Runs | Trust |
|---|---|---|
| `portfolio` | The portfolio site | Production workload |
| `cloudflare` | The Cloudflare Tunnel connector | Infrastructure — outbound-only, no inbound ports |
| `arc-systems` | The ARC controller and listener | High: installed software that can create pods |
| `arc-runners` | Ephemeral runner pods, one per CI job | Low: runs whatever code a workflow contains |
| `flux-system` | Flux's own controllers, plus every `Kustomization`/`ImageRepository`/`ImagePolicy`/`ImageUpdateAutomation` CR | Highest: can write to any namespace in the cluster |

The controller and the runners are kept in separate namespaces so RBAC/quotas can target the low-trust runner
pods without touching the controller.

## How ARC is installed

Both the controller and the `pi-k3s` runner scale set are `HelmRelease` objects using
`chart.kind: OCIRepository` (the current Flux-recommended way to deploy an OCI Helm chart — no separate
`OCIRepository` object needed, it's inline). Both pin the exact same chart version. `pi-k3s`'s `HelmRelease` has
`dependsOn` the controller's, so helm-controller won't install the scale set before the controller's CRDs exist.

`releaseName` on both exactly matches the Helm release names from when these were installed by hand
(`arc`, `pi-k3s`) — that's what let Flux adopt the existing installs in place instead of creating duplicates.

## Image automation

- **`ImageRepository`** scans `snayeem/portfolio-app` on Docker Hub.
- **`ImagePolicy`** picks the tag with the highest trailing run number (`filterTags.pattern` extracts the
  `-<run#>` suffix from CI's `sha-<hash>-<run#>` tags; a plain `sha-<hash>` tag has no sortable component, which
  is why CI's tag format includes the run number at all).
- **`ImageUpdateAutomation`** commits the new tag back into `apps/portfolio/deployment.yaml` using the
  `Setters` strategy (the `# {"$imagepolicy": "flux-system:portfolio-app"}` marker comment on the `image:` line).

`deployment.yaml`'s image tag is actively rewritten by these commits (author: `fluzcdbot`) — it's the live source
of truth for what's deployed. Don't hand-edit that tag.

## The Cloudflare Tunnel

`cloudflared` runs as a `Deployment` in its own `cloudflare` namespace and makes an outbound-only connection to
Cloudflare's edge. Hostname routing (which public hostname maps to which in-cluster Service) is configured in
the Cloudflare dashboard under Zero Trust → Networks → Tunnels → Public Hostname, pointing at
`portfolio-web.portfolio.svc.cluster.local:9123` — the fully-qualified Service name, since the tunnel connector
and the app it routes to live in different namespaces.

## Security model

- **Ephemeral runners.** Every CI job gets a fresh pod, deleted afterward.
- **No Kubernetes credentials in CI.** The `aws-resume` CI workflow only builds and pushes an image — it never
  touches the cluster. Flux does the entire deploy side, authenticated with its own Git deploy key.
- **GitHub App, not a personal token**, for ARC. Installed on the `aws-resume` repo only, with
  Administration (read/write) so it can register runners. Tokens expire hourly.
- **Public repo.** The deploy job concept doesn't exist anymore; pull request workflows don't reach the cluster
  and fork PRs need approval before running.
- **No inbound ports.** Both cloudflared and the ARC listener only make outbound connections.

## One-time setup

### k3s + Flux

```bash
grep -w memory /sys/fs/cgroup/cgroup.controllers   # kubernetes needs the memory cgroup
sudo modprobe vxlan                                # pod networking needs vxlan

# traefik and servicelb are disabled — Cloudflare Tunnel handles ingress, nothing is exposed on the LAN
curl -sfL https://get.k3s.io | sh -s - --disable traefik --disable servicelb

mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER": ~/.kube/config && chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config
```

Install the `flux` CLI, then bootstrap it against this repo at `clusters/pi` (see
[Flux's bootstrap docs](https://fluxcd.io/flux/installation/bootstrap/github/) for the exact command/flags —
worth checking current docs rather than copying an old command verbatim, since this project has already hit a
couple of stale-API surprises from trusting memory over the current docs).

## Day-to-day commands

```bash
flux get kustomizations
flux get helmreleases -A
kubectl -n flux-system get imagerepository,imagepolicy,imageupdateautomation
kubectl -n portfolio get deploy portfolio-web -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
kubectl -n portfolio get pods
kubectl -n arc-runners get pods -w                        # watch runners come and go
kubectl -n arc-systems logs deploy/arc-gha-rs-controller   # if runners don't start
kubectl -n cloudflare logs -f deploy/cloudflare            # tunnel connector status
```

## Known limitations

- **One node, one replica.** A Pi reboot takes the site down; no redundancy against hardware failure.
- **Secrets are created by hand**, stored as base64-encoded Kubernetes Secrets, not a vault.
- **The runner image uses `:latest`** (`ghcr.io/actions/actions-runner`) on purpose — GitHub stops accepting
  runners that are too far out of date. The ARC chart version itself *is* pinned.

## Problems hit while building this

| Symptom | Cause |
|---|---|
| `no matches for kind "ImageRepository" in version "...v1beta2"` | Wrote CRD YAML from memory with a stale `apiVersion`. Check `kubectl api-resources \| grep toolkit.fluxcd.io` against what's actually installed before trusting a remembered version. |
| Image automation objects silently landed in the wrong namespace | A Flux `Kustomization`'s `targetNamespace` overrides **every** resource's namespace unconditionally, even one with an explicit `metadata.namespace` already set. Move anything needing a different namespace out of that `Kustomization`'s `path`. |
| `ImageUpdateAutomation` stuck `Stalled: True`, `RemovedTemplateField` | `messageTemplate` used `.Updated.Images`, a field removed from current Flux. Use `.Changed.Summary` instead. |
| `helm list -A` showed a duplicate release after adopting ARC into Flux | `HelmRelease.spec.releaseName` didn't match the existing Helm release name. It must match exactly for Flux to adopt in place rather than create a new one. |
| cloudflared tunnel stopped reaching the site after moving namespaces | The dashboard's route used a bare short hostname, which only resolves within the pod's own namespace's DNS search list. Needs the fully-qualified Service name once tunnel and target Service are in different namespaces. |
