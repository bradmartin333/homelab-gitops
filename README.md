# homelab-gitops

GitOps repo for the homelab Kubernetes cluster. Flux watches this repo and
reconciles the cluster to match it — cluster changes flow through git commits,
not `kubectl apply`.

## Context

Learning Kubernetes by migrating self-hosted internal tools from ad-hoc
Docker containers (accessed by VPNing into the host via Tailscale and hitting
ports) to a k8s cluster managed via GitOps. A single app wouldn't justify
k8s; a multi-app homelab does. Dockge was considered and ruled out before
this (Docker-compose-specific, doesn't apply on k8s).

## Services

- **Vikunja** — task management (the original motivating app)
- **tldraw** — whiteboard (stateless; good low-risk ingress practice)
- **Grafana** + **Prometheus** — dashboards + metrics (monitoring the cluster
  itself is an early win)
- **PostgreSQL** (shared, general-purpose) + **Adminer** — DB + web admin UI
- **Git server** (Gitea/Forgejo) — will also host custom apps that talk to
  the other services
- **Splash page** — reachable on plain :80 (e.g. `http://homelab/`), links to
  everything else
- **Flux image automation** — the Watchtower replacement: updates image tags
  in this repo via git commits, Flux applies them (cluster state always flows
  through git)

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

**Networking model:** Tailscale operator and Traefik serve different purposes
and both are used. The operator gives individual Services their own tailnet
identity/MagicDNS name for secure remote access (replacing the old "VPN into
the host, hit a port" pattern); Traefik handles LAN-friendly routing and the
plain-:80 splash page.

**Day-to-day cluster poking** (the Dockge-replacement role, separate from
GitOps): try k9s (terminal) and/or Headlamp; Lens and Portainer are the other
candidates. Not yet decided.

## Still open

- [ ] Whether the splash page is LAN-only via Traefik, or also exposed on the
      tailnet
- [ ] Whether custom apps (hosted on the self-hosted git server) get built as
      container images automatically (CI) or manually for now
- [ ] k9s vs Headlamp vs Lens vs Portainer for day-to-day visualization

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
