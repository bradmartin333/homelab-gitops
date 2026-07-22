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

**Web UIs on the tailnet use HTTPS** via a `tailscale`-class **Ingress** (not a
plain LoadBalancer Service) — the operator issues a real Let's Encrypt
`*.ts.net` cert, so browsers load it with no warning (plain HTTP breaks under
Chrome's HTTPS-First Mode). Requires HTTPS certs enabled once at
admin console → DNS → HTTPS Certificates. Pattern (see
`apps/postgres-adminer/adminer.yaml`): a ClusterIP Service for the app + an
Ingress with `ingressClassName: tailscale` and `tls.hosts: [<bare-label>]`;
the app then lives at `https://<bare-label>.<tailnet>.ts.net`. Use this for
every web UI (Adminer, Vikunja, Grafana, …).

> Testing note: the Pi node joined with `--accept-dns=false`, so *the Pi
> itself* can't resolve `*.ts.net` MagicDNS names or route to tailnet peers —
> curling a service from the Pi fails/times out even when it's healthy. Test
> from a full tailnet client (your Mac/phone) instead. The TLS cert can still
> be probed directly: `openssl s_client -connect <peer-ip>:443 -servername …`.

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

## Secrets (SOPS + age)

Secrets are committed to git **encrypted** with [SOPS](https://github.com/getsops/sops)
using an [age](https://github.com/FiloSottile/age) key. Flux decrypts them in
the cluster at apply time. Only `data`/`stringData` values are encrypted; k8s
metadata (name, namespace) stays readable in git for review.

> ⚠️ **The age private key is the master key to every secret in this repo.**
> It lives in two places and is **never** committed:
> - `~/.config/sops/age/keys.txt` on the admin workstation (back this up
>   somewhere safe — a password manager — or you can never decrypt again)
> - the `sops-age` secret in the `flux-system` namespace (how Flux decrypts)
>
> Public key (safe, it's in `.sops.yaml`):
> `age1rectewyjyw259ks6e8epgac022665n88yf9ep64fe7kky92thgkq8tfs7e`

One-time setup (already done):
```sh
brew install sops age
age-keygen -o ~/.config/sops/age/keys.txt          # generate keypair
# load private key into the cluster for Flux to decrypt with:
cat ~/.config/sops/age/keys.txt | kubectl create secret generic sops-age \
  -n flux-system --from-file=age.agekey=/dev/stdin
# .sops.yaml pins the public key; each Flux Kustomization has a
# `decryption: { provider: sops, secretRef: { name: sops-age } }` block.
```

**Adding an encrypted secret** (the recurring workflow):
```sh
# sops doesn't auto-find the key at ~/.config/sops/age/keys.txt on macOS —
# export this once (add to your shell profile) so encrypt/edit/decrypt work:
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
# 1. write the Secret manifest as plaintext, filename ending in .sops.yaml
# 2. encrypt in place (sops picks up rules from .sops.yaml):
sops --encrypt --in-place apps/<app>/secret.sops.yaml
# 3. add it to that app's kustomization.yaml resources, commit. Done.
# To edit later: `sops apps/<app>/secret.sops.yaml` (decrypts, re-encrypts on save).
```

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
     - homelab.tail87ca97.ts.net   # Tailscale MagicDNS (added in step, below)
     - 192.168.1.99                # eth0 DHCP address
     - 192.168.1.100               # prior wlan0 address (fallback)
   EOF'
   ssh brad@homelab.lan 'curl -sfL https://get.k3s.io | sh -'
   ```
   > As built: took `stable`, which resolved to **v1.36.2+k3s1** (arm64).
   >
   > Heads up on the node IP: it's DHCP and **changes with the interface** —
   > the Pi came up on `.100` over Wi-Fi, then `.99` once moved to Ethernet.
   > Connecting by *name* (`homelab.lan` / the tailnet name) sidesteps this;
   > for stability, set a DHCP reservation on the router. The Tailscale
   > MagicDNS name above was added later (see the Tailscale step) and is the
   > only address that also works off-LAN.
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

   **Phone / off-LAN:** `homelab.lan` only resolves on the LAN. Use the
   Tailscale MagicDNS name instead (set up in step 5) as the kubeconfig server
   address — it's already in `tls-san`. See that step for details.

5. **Install Tailscale on the node** (distinct from the in-cluster Tailscale
   *operator* in step 7: this gives the **Pi itself** a stable tailnet
   identity, so SSH and the k8s API work from off-LAN — e.g. your phone —
   regardless of LAN DHCP). Auth is interactive: run `up`, open the printed
   URL, approve the device.
   ```sh
   ssh brad@homelab.lan 'curl -fsSL https://tailscale.com/install.sh | sh'
   ssh brad@homelab.lan 'sudo tailscale up --hostname=homelab --accept-dns=false'
   # --accept-dns=false keeps the Pi's own resolver (SSH by homelab.lan still
   # works). Grab the assigned MagicDNS name:
   ssh brad@homelab.lan 'sudo tailscale status --json' | grep -o '"DNSName": *"homelab[^"]*"'
   ```
   > As built: node joined as `homelab` → `homelab.tail87ca97.ts.net`
   > (tailnet IP `100.92.53.11`). This name was added to `tls-san` in step 3
   > (cert regenerated). To use it from a phone/off-LAN device, set the
   > kubeconfig server to `https://homelab.tail87ca97.ts.net:6443`. A mobile
   > kubectl/dashboard app on the tailnet then reaches the cluster from
   > anywhere.

6. **Install the Flux CLI** on your workstation: `brew install fluxcd/tap/flux`
   (as built: v2.9.2). Pre-flight: `flux check --pre`.
7. **Bootstrap Flux** against this repo (creates `clusters/home/flux-system/`
   and commits it). Flux needs a GitHub token with `repo` scope; easiest is to
   reuse the `gh` CLI login:
   ```sh
   brew install gh          # if needed
   gh auth login            # GitHub.com → HTTPS → browser; grants `repo` scope
   export GITHUB_TOKEN=$(gh auth token)
   flux bootstrap github \
     --owner=bradmartin333 \
     --repository=homelab-gitops \
     --branch=main \
     --path=clusters/home \
     --personal              # repo is under a personal account (private)
   ```
8. **Tailscale operator secret** — the in-cluster operator (distinct from the
   node's own tailscale login in step 5) authenticates to your tailnet with an
   **OAuth client**. Until the `operator-oauth` secret exists, the operator pod
   sits in `ContainerCreating` with `FailedMount ... secret "operator-oauth"
   not found`, so the `infrastructure` Kustomization stays "in progress"
   (it has `wait: true`).

   a. **Access Controls** (ACL editor) → add tag owners, Save:
   ```json
   "tagOwners": {
     "tag:k8s-operator": ["autogroup:admin"],
     "tag:k8s":          ["tag:k8s-operator"]
   }
   ```
   `tag:k8s-operator` is the operator's own identity; `tag:k8s` is what it
   applies to the Service proxy devices it creates.

   b. **Settings → OAuth clients → Generate** — scope `Devices: Core` **Write**
   (grants the auth-keys write it needs), tag `tag:k8s-operator`. Copy the
   Client ID and Secret.

   c. Create the secret (namespace already exists, made by Flux). The operator
   is in a mount-retry loop, so it picks this up within ~1m — no restart:
   ```sh
   kubectl create secret generic operator-oauth -n tailscale \
     --from-literal=client_id='<id>' --from-literal=client_secret='<secret>'
   ```
   (Move this into git via SOPS once secrets management is set up — until then
   it's the one piece of cluster state not in git.)
9. Watch it converge: `flux get kustomizations --watch`
   (`infrastructure` → Ready once the operator is up, then `apps` applies.)

   Two snags we hit here, both recoverable:
   - **Operator `403: calling actor does not have enough permissions`** when
     minting its auth key → the OAuth client's tag wasn't valid yet. Fix the
     ACL `tagOwners` (step 8a) and ensure the client has `Auth Keys: Write` +
     tag `tag:k8s-operator`; the operator retries on pod restart. Check with
     `kubectl logs -n tailscale -l app=operator`.
   - **HelmRelease stuck `Failed` / "timeout waiting for Deployment"** — while
     the operator crash-looped on the 403, the Helm install's 5m timer expired
     and marked the release failed, even though the Deployment recovered after.
     Clear the stale state with a forced retry:
     `flux reconcile helmrelease tailscale-operator -n tailscale --force`.

## Phased plan

- [x] 1. k3s up on the Pi (v1.36.2+k3s1; kubectl works from the Mac)
- [x] 2. Flux bootstrapped (v2.9.2); tailscale operator reconciled & on the tailnet
- [x] 3. Splash page reachable on :80 via Traefik (`curl http://homelab.lan/` → 200)
- [x] 4. Postgres + Adminer — StatefulSet + 5Gi local-path PVC; SOPS-encrypted
      credentials; Adminer tailnet-only at `postgres-adminer-ts.tail87ca97.ts.net`
- [x] 5. Vikunja (2.4.0) → dedicated db/role on shared Postgres; SOPS secrets;
      HTTPS at `vikunja.tail87ca97.ts.net` (37 tables migrated OK)
- [x] 6. Grafana + Prometheus — kube-prometheus-stack; 11 scrape targets up;
      Grafana HTTPS at `grafana.tail87ca97.ts.net` (SOPS admin secret)
- [x] 7. tldraw — self-built persistent+multiplayer sync server
      (bradmartin333/node-tldraw:2.0.0), SQLite on a 2Gi PVC, HTTPS at
      `tldraw.tail87ca97.ts.net`. First custom app in the cluster.
- [ ] 8. Self-hosted git server (Gitea/Forgejo); maybe migrate this repo into it
- [ ] 9. Flux image automation (Watchtower replacement)
