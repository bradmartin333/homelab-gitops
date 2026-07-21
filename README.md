# homelab-gitops

GitOps repo for the homelab Kubernetes cluster. Flux watches this repo and
reconciles the cluster to match it — cluster changes flow through git commits,
not `kubectl apply`.

## Decisions

| Topic | Decision |
|---|---|
| Cluster | k3s, single node, on a Raspberry Pi (Raspberry Pi OS Lite 64-bit) |
| GitOps controller | Flux |
| Ingress | Traefik (bundled with k3s) |
| Storage | local-path-provisioner (bundled with k3s) |
| Image updates | Flux image automation (added last, per plan) |
| Repo home | GitHub for bootstrap; may migrate to self-hosted Gitea/Forgejo later |
| Remote access | Tailscale operator — Services get their own tailnet MagicDNS names |
| Secrets | SOPS + age (Flux-native); set up when the first secret is needed (Tailscale OAuth creds) |

## Layout

```
clusters/home/        Flux entrypoint: flux-system (created by bootstrap) +
                      Kustomization CRs pointing at the paths below
infrastructure/       cluster-wide concerns: tailscale operator, namespaces, ...
apps/                 the actual services, one directory each
```

Flux applies `infrastructure/` first, then `apps/` (declared via `dependsOn`).

## Bootstrap runbook

The cluster host is a Raspberry Pi; `kubectl`/`flux` run from your workstation.

1. **Provision the Pi** — Raspberry Pi OS Lite (64-bit) via Raspberry Pi
   Imager. In Imager's settings, set hostname (`homelab`), user, and SSH
   public key; use Ethernet. Boot from USB SSD if possible (SD cards wear out
   under Postgres/Prometheus write load). Then enable the memory cgroup
   (required by k3s): append `cgroup_memory=1 cgroup_enable=memory` to the
   line in `/boot/firmware/cmdline.txt` and reboot.
2. **Install k3s** (Traefik + local-path storage included):
   ```sh
   curl -sfL https://get.k3s.io | sh -
   # copy /etc/rancher/k3s/k3s.yaml to ~/.kube/config on your workstation,
   # replacing 127.0.0.1 with the host's address
   ```
3. **Install the Flux CLI** on your workstation: `brew install fluxcd/tap/flux`
4. **Bootstrap Flux** against this repo (creates `clusters/home/flux-system/`
   and commits it):
   ```sh
   export GITHUB_TOKEN=<personal access token with repo scope>
   flux bootstrap github \
     --owner=<github-username> \
     --repository=homelab-gitops \
     --branch=main \
     --path=clusters/home \
     --personal
   ```
5. **Tailscale operator secret** — create an OAuth client in the Tailscale
   admin console (scopes: `devices:write`, `auth_keys:write`; tag
   `tag:k8s-operator`), then:
   ```sh
   kubectl create secret generic operator-oauth -n tailscale \
     --from-literal=client_id=<id> --from-literal=client_secret=<secret>
   ```
   (Move this into git via SOPS once secrets management is set up.)
6. Watch it converge: `flux get kustomizations --watch`

## Phased plan

- [ ] 1. k3s up on the Pi
- [ ] 2. Flux bootstrapped; tailscale operator reconciled, one test Service on the tailnet
- [ ] 3. Splash page reachable on :80 via Traefik
- [ ] 4. Postgres + Adminer (first real app: PVCs + Secrets)
- [ ] 5. Vikunja, pointed at shared Postgres
- [ ] 6. Grafana + Prometheus
- [ ] 7. tldraw
- [ ] 8. Self-hosted git server (Gitea/Forgejo); maybe migrate this repo into it
- [ ] 9. Flux image automation (Watchtower replacement)
