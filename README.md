# node-infra

Server layer of a small self-hosted container platform on k3s. This repository describes everything
on the node below Kubernetes and is applied with Ansible.

## Where it fits

The platform has three layers. Each one has one kind of repository that describes it and one mechanism
that applies it, and each layer pulls its own state from `main`:

| Layer | Repository | Applied by |
| --- | --- | --- |
| Tenant apps | one app repo per project (code, Dockerfile, `deploy/` Helm chart) | Argo CD, after CI pushed an image tagged with the commit SHA |
| Platform | GitOps repo for the cluster (Argo CD, Traefik, cert-manager, monitoring, CI runners, tenant list) | Argo CD, within minutes of a merge |
| **Server** | **node-infra** (this repo) | `ansible-pull` timer on the node, within 15 minutes of a merge |

## Principles

- **Only `main` counts.** The node applies nothing but `main`. A merged pull request is the release approval.
- **Pull, not push.** No CI job gets access to the node. The node fetches and applies changes itself.
- **Bots open pull requests, humans merge.** Nobody pushes to `main` directly.
- **No secrets and no per-node values in git.** This repository is public. Addresses, hostnames, keys and
  tokens stay on the node, e.g. in `/etc/node-infra/pull.env`.

## What belongs here

| Belongs here | Does not belong here |
| --- | --- |
| Everything below Kubernetes: OS packages and updates, SSH, VPN and DNS services on the host, k3s installation and configuration, backups, and once the bootstrap of Argo CD so the cluster manages itself afterwards. | Kubernetes manifests of apps or platform services. After the bootstrap, Ansible does not touch anything inside the cluster. |

## Status

The playbook manages the `ansible-pull` timer, Docker, wg-easy (WireGuard VPN with web UI, in Docker) and
dnsmasq (split DNS for VPN clients), k3s and the wg-easy web UI behind Traefik with a Let's
Encrypt certificate, reachable only through the VPN. Planned roles, each in its own pull request: `base`
(packages, updates), `argocd-bootstrap`.

## Layout

```text
ansible.cfg
inventory.yml          localhost only (ansible-pull runs on the node)
site.yml               entry point for ansible-pull
pull.env.example       sample for /etc/node-infra/pull.env on the node
group_vars/all.yml     maps the environment variables to Ansible variables
roles/
  ansible_pull/        installs ansible-core and git, the service and the timer
  docker/              Docker and the compose plugin from the Ubuntu archive
  wg_easy/             compose file in /etc/docker/containers/wg-easy, UI only on the bridge address
  dnsmasq/             split DNS, listens only on the bridge address
  k3s/                 k3s in a pinned version, API also valid for the bridge address
  vpn_ui/              cert-manager, Let's Encrypt issuer, Traefik route to the wg-easy UI (VPN only)
```

## Environment on the node

`/etc/node-infra/pull.env`, written by hand from [`pull.env.example`](pull.env.example), read by
`node-infra-pull.service`:

| Variable | Meaning |
| --- | --- |
| `NODE_INFRA_REPO_URL` | Repository to pull, e.g. `https://github.com/gpt-partners/node-infra.git` |
| `NODE_INFRA_BRANCH` | Branch to apply, `main` |
| `NODE_INFRA_VPN_SUBNET_IPV4`, `NODE_INFRA_VPN_SUBNET_IPV6` | Subnets of the wg-easy Docker bridge |
| `NODE_INFRA_VPN_BRIDGE_IPV4` | Host address on that bridge (first address of the IPv4 subnet). The wg-easy UI and dnsmasq listen only here. |
| `NODE_INFRA_WG_EASY_IPV4`, `NODE_INFRA_WG_EASY_IPV6` | Fixed addresses of the wg-easy container |
| `NODE_INFRA_DNS_VPN_DOMAINS` | Names that resolve to the bridge address for VPN clients, comma-separated |
| `NODE_INFRA_DNS_UPSTREAMS` | Resolvers for everything else, comma-separated |
| `NODE_INFRA_VPN_UI_HOST` | Name of the wg-easy web UI. Public DNS points it to the node (HTTP-01); VPN clients resolve it to the bridge address. |
| `NODE_INFRA_ACME_EMAIL` | Let's Encrypt account e-mail |

The playbook stops with a clear error if the file or a value is missing. Add new values to the file
on the node before merging the pull request that needs them.

## First run (once, by hand, as root on the node)

```bash
apt-get update && apt-get install -y ansible-core git

# Start from the sample in this repo, then fill in the real values.
install -d -m 0755 /etc/node-infra
curl -fsSL https://raw.githubusercontent.com/gpt-partners/node-infra/main/pull.env.example \
  -o /etc/node-infra/pull.env
chmod 0644 /etc/node-infra/pull.env
nano /etc/node-infra/pull.env

set -a; . /etc/node-infra/pull.env; set +a
ansible-pull --url "$NODE_INFRA_REPO_URL" --checkout "$NODE_INFRA_BRANCH" \
  --directory /var/lib/node-infra/checkout site.yml
```

From then on the timer takes over (first run 5 minutes after boot, then every 15 minutes).

## Check

```bash
systemctl list-timers node-infra-pull.timer      # next and last run
journalctl -u node-infra-pull.service -n 50      # output of the last runs
systemctl start node-infra-pull.service          # pull now instead of waiting

. /etc/node-infra/pull.env
dig +short @"$NODE_INFRA_VPN_BRIDGE_IPV4" "${NODE_INFRA_DNS_VPN_DOMAINS%%,*}"   # the bridge address
dig +short @"$NODE_INFRA_VPN_BRIDGE_IPV4" example.com                         # a public address
ss -ltn | grep 51821                             # only on the bridge address, never 0.0.0.0
k3s kubectl get nodes                            # Ready
k3s kubectl -n wg-easy get certificate           # wg-easy-tls  READY True
curl -s -o /dev/null -w '%{http_code}\n' "https://$NODE_INFRA_VPN_UI_HOST/"   # 403 from the internet, 200 or 302 through the VPN
```

A changed compose file recreates the wg-easy container: the tunnel drops for a few seconds. The
previous file is kept as `docker-compose.yml.<timestamp>~`.

## Limits

- `--only-if-changed`: a run that fails is not retried until the next commit on `main`. To retry
  without a commit: `rm -rf /var/lib/node-infra/checkout && systemctl start node-infra-pull.service`.
- Whoever can change `main` gets root on the node. Protect `main` (ruleset: pull request required, no
  force push, no bypass for bots).
- Stop applying changes: `systemctl disable --now node-infra-pull.timer`.

## Local checks

```bash
ansible-playbook --syntax-check site.yml
ansible-lint
```

CI runs the same two commands on every pull request.
