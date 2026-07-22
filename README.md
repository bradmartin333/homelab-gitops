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
   public key; use Ethernet. Boot from USB SSD (SD cards wear out under
   Postgres/Prometheus write load).

   > As built: Pi 4 (8GB), 256GB external SSD as the boot/root drive,
   > reachable at `brad@homelab.lan`. Verify with:
   > `ssh brad@homelab.lan 'uname -m'` → should print `aarch64`.

2. **Enable the memory cgroup** (required by k3s; Pi OS ships with it off).
   Append the two params to the **single line** in
   `/boot/firmware/cmdline.txt` — a stray
   newline breaks boot, so back it up and append in place rather than editing
   by hand, then reboot:
   ```sh
   ssh brad@homelab.lan '
     sudo cp /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.bak &&
     sudo sed -i "s/[[:space:]]*$/ cgroup_memory=1 cgroup_enable=memory/" /boot/firmware/cmdline.txt &&
     cat /boot/firmware/cmdline.txt
   '
   # cmdline.txt has no trailing newline, so `wc -l` reads 0 — that is fine.
   # What matters: it is still ONE physical line with the two params at the end.
   ssh brad@homelab.lan 'sudo reboot'
   # after it comes back (~25s), confirm the memory controller is enabled.
   # Pi OS uses cgroup v2, so check the unified hierarchy, NOT /proc/cgroups
   # (which never lists memory under v2 and will look like it failed):
   ssh brad@homelab.lan 'cat /sys/fs/cgroup/cgroup.controllers'
   # → must include `memory`, e.g. "cpuset cpu io memory pids"
   ```
3. **Install k3s** (Traefik + local-path storage included). Before installing,
   drop a config file so the API server's serving cert covers the names you'll
   connect by — otherwise `kubectl` from other machines fails TLS verification
   (the default cert is only valid for `homelab`, `localhost`, etc.):
   ```sh
   ssh brad@homelab.lan 'sudo tee /etc/rancher/k3s/config.yaml >/dev/null <<EOF
   tls-san:
     - homelab
     - homelab.lan
     - 192.168.1.100
     # add the Tailscale MagicDNS name here once the tailnet is up, e.g.
     # - homelab.<tailnet>.ts.net
   EOF'
   ssh brad@homelab.lan 'curl -sfL https://get.k3s.io | sh -'
   ```
   > As built: took `stable`, which resolved to **v1.36.2+k3s1** (arm64).
   >
   > If you edit `tls-san` *after* k3s is already running, the cert won't
   > regenerate on its own — delete it and restart so it's rebuilt:
   > ```sh
   > ssh brad@homelab.lan 'sudo rm -f \
   >   /var/lib/rancher/k3s/server/tls/serving-kube-apiserver.crt \
   >   /var/lib/rancher/k3s/server/tls/serving-kube-apiserver.key \
   >   && sudo systemctl restart k3s'
   > ```

   Sanity check on the host:
   ```sh
   ssh brad@homelab.lan 'sudo k3s kubectl get nodes'   # STATUS Ready
   ```

4. **Get cluster access on your devices.** k3s writes an admin kubeconfig at
   `/etc/rancher/k3s/k3s.yaml` on the Pi, pointing at `127.0.0.1`. For any
   remote machine, copy it and rewrite the server address to a name the cert
   covers (a `tls-san` from step 3).

   **macOS / Linux workstation** (what this repo was set up from):
   ```sh
   mkdir -p ~/.kube
   [ -f ~/.kube/config ] && cp ~/.kube/config ~/.kube/config.bak.$(date +%s)
   ssh brad@homelab.lan 'sudo cat /etc/rancher/k3s/k3s.yaml' \
     | sed -e 's#https://127.0.0.1:6443#https://homelab.lan:6443#' \
           -e 's/: default/: homelab/g' -e 's/name: default/name: homelab/g' \
     > ~/.kube/config
   chmod 600 ~/.kube/config
   kubectl get nodes           # confirm reachable
   ```
   To keep multiple clusters, write to a separate file and merge instead:
   `KUBECONFIG=~/.kube/config:~/.kube/homelab.yaml kubectl config view --flatten`.

   **Windows** — same idea; the config lives at `%USERPROFILE%\.kube\config`.
   Pull the file (e.g. `scp brad@homelab.lan:...` via sudo, or copy it out),
   and replace `https://127.0.0.1:6443` with `https://homelab.lan:6443`.

   **Phone / off-LAN, via Tailscale:** `homelab.lan` only resolves on the LAN.
   Once the Tailscale operator is up (step 6) the *cluster's* Services get
   tailnet names, but to reach the *API server* from your phone, also install
   Tailscale on the Pi itself so the node has a stable MagicDNS name
   (`homelab.<tailnet>.ts.net`). Add that name to `tls-san` (step 3), then use
   it as the server address in the kubeconfig. Mobile kubectl apps (e.g.
   Kubernetes-native dashboards, or a Headlamp/k9s session over SSH) then work
   from anywhere on the tailnet.

5. **Install the Flux CLI** on your workstation: `brew install fluxcd/tap/flux`
6. **Bootstrap Flux** against this repo (creates `clusters/home/flux-system/`
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
7. **Tailscale operator secret** — create an OAuth client in the Tailscale
   admin console (scopes: `devices:write`, `auth_keys:write`; tag
   `tag:k8s-operator`), then:
   ```sh
   kubectl create secret generic operator-oauth -n tailscale \
     --from-literal=client_id=<id> --from-literal=client_secret=<secret>
   ```
   (Move this into git via SOPS once secrets management is set up.)
8. Watch it converge: `flux get kustomizations --watch`

## Phased plan

- [x] 1. k3s up on the Pi (v1.36.2+k3s1; kubectl works from the Mac)
- [ ] 2. Flux bootstrapped; tailscale operator reconciled, one test Service on the tailnet
- [ ] 3. Splash page reachable on :80 via Traefik
- [ ] 4. Postgres + Adminer (first real app: PVCs + Secrets)
- [ ] 5. Vikunja, pointed at shared Postgres
- [ ] 6. Grafana + Prometheus
- [ ] 7. tldraw
- [ ] 8. Self-hosted git server (Gitea/Forgejo); maybe migrate this repo into it
- [ ] 9. Flux image automation (Watchtower replacement)
