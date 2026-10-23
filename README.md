# server

Server layer of a small self-hosted container platform on k3s. This repository describes everything
on the node below Kubernetes and is applied with Ansible.

## Where it fits

The platform has three layers. Each one has one kind of repository that describes it and one mechanism
that applies it, and each layer pulls its own state from `main`:

| Layer | Repository | Applied by |
| --- | --- | --- |
| Tenant apps | one app repo per project (code, Dockerfile, `deploy/` Helm chart) | Argo CD, after CI pushed an image tagged with the commit SHA |
| Platform | **platform** (private, GitOps repo for the cluster: Argo CD, Traefik, cert-manager, monitoring, CI runners, tenant list) | Argo CD, within minutes of a merge |
| **Server** | **server** (this repo) | `ansible-pull` timer on the node, within 15 minutes of a merge |

This repository used to be called `node-infra`. The names on the node (`/etc/node-infra`, `NODE_INFRA_*`,
`node-infra-pull.service`) keep that name, so a running node needs no migration.

## Principles

- **Only `main` counts.** The node applies nothing but `main`. A merged pull request is the release approval.
- **Pull, not push.** No CI job gets access to the node. The node fetches and applies changes itself.
- **Bots open pull requests, humans merge.** Nobody pushes to `main` directly.
- **No secrets and no per-node values in git.** This repository is public. Addresses, hostnames, keys and
  tokens stay on the node, e.g. in `/etc/node-infra/pull.env`.

## What belongs here

| Belongs here | Does not belong here |
| --- | --- |
| Everything below Kubernetes: OS packages and updates, SSH, VPN and DNS services on the host, k3s installation and configuration, backups, and once the bootstrap of Argo CD so the cluster manages itself afterwards. | Kubernetes manifests of apps or platform services. After the bootstrap, Ansible touches only the Services that point to the host (`host_services`). |

## Status

The playbook manages the `ansible-pull` timer, security updates with automatic reboots, key-only
SSH, Docker, wg-easy (WireGuard VPN with web UI, in Docker),
dnsmasq (split DNS for VPN clients) and k3s. It publishes the wg-easy web UI into the cluster as a
Service; the route with certificate and the VPN-only rule are in the `platform` repository. Argo CD is
installed once, then manages itself and everything else from the private `platform` repository
(app of apps, `platform/argocd.yaml`).

## Layout

```text
ansible.cfg
inventory.yml          localhost only (ansible-pull runs on the node)
site.yml               entry point for ansible-pull
pull.env.example       sample for /etc/node-infra/pull.env on the node
group_vars/all.yml     maps the environment variables to Ansible variables
roles/
  ansible_pull/        installs ansible-core and git, the service and the timer
  base/                daily security updates with automatic reboot, SSH with keys only
  docker/              Docker and the compose plugin from the Ubuntu archive
  wg_easy/             compose file in /etc/docker/containers/wg-easy, UI only on the bridge address
  dnsmasq/             split DNS, listens only on the bridge address
  k3s/                 k3s in a pinned version, API also valid for the bridge address
  host_services/       Service and EndpointSlice for the wg-easy UI on the host, nothing else in the cluster
  argocd_bootstrap/    first install of Argo CD, repository credentials from a key file, root application
  backup/              restic in a pinned version, nightly backup to Backblaze B2, weekly check
```

## Environment on the node

`/etc/node-infra/pull.env`, written by hand from [`pull.env.example`](pull.env.example), read by
`node-infra-pull.service`:

| Variable | Meaning |
| --- | --- |
| `NODE_INFRA_REPO_URL` | Repository to pull, e.g. `https://github.com/gpt-partners/server.git` |
| `NODE_INFRA_BRANCH` | Branch to apply, `main` |
| `NODE_INFRA_VPN_SUBNET_IPV4`, `NODE_INFRA_VPN_SUBNET_IPV6` | Subnets of the wg-easy Docker bridge |
| `NODE_INFRA_VPN_BRIDGE_IPV4` | Host address on that bridge (first address of the IPv4 subnet). The wg-easy UI and dnsmasq listen only here. |
| `NODE_INFRA_WG_EASY_IPV4`, `NODE_INFRA_WG_EASY_IPV6` | Fixed addresses of the wg-easy container |
| `NODE_INFRA_DNS_VPN_DOMAINS` | Names that resolve to the bridge address for VPN clients, comma-separated |
| `NODE_INFRA_DNS_UPSTREAMS` | Resolvers for everything else, comma-separated |
| `NODE_INFRA_ARGOCD_REPO_URL` | Platform repository Argo CD follows (SSH URL) |
| `NODE_INFRA_ARGOCD_DEPLOY_KEY_FILE` | File on the node with the read-only deploy key for that repository |
| `NODE_INFRA_ARGOCD_GITHUB_APP_ID`, `NODE_INFRA_ARGOCD_GITHUB_APP_INSTALLATION_ID`, `NODE_INFRA_ARGOCD_GITHUB_APP_KEY_FILE` | Optional, all three or none: GitHub App (Contents: read only) with which Argo CD reads the app repositories of the same account over HTTPS |

The playbook stops with a clear error if the file or a value is missing. Add new values to the file
on the node before merging the pull request that needs them.

## First run (once, by hand, as root on the node)

```bash
apt-get update && apt-get install -y ansible-core git

# Start from the sample in this repo, then fill in the real values.
install -d -m 0755 /etc/node-infra
curl -fsSL https://raw.githubusercontent.com/gpt-partners/server/main/pull.env.example \
  -o /etc/node-infra/pull.env
chmod 0644 /etc/node-infra/pull.env
nano /etc/node-infra/pull.env

set -a; . /etc/node-infra/pull.env; set +a
ansible-pull --url "$NODE_INFRA_REPO_URL" --checkout "$NODE_INFRA_BRANCH" \
  --directory /var/lib/node-infra/checkout site.yml
```

### Deploy key for Argo CD (once)

```bash
ssh-keygen -t ed25519 -N '' -C argocd-read-only -f /etc/node-infra/argocd-deploy-key
chmod 0600 /etc/node-infra/argocd-deploy-key
cat /etc/node-infra/argocd-deploy-key.pub
```

Add the public key on GitHub to the `platform` repository: Settings → Deploy keys → Add deploy key,
title `argocd (read-only)`, **without** write access. The private key never leaves the node.

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
k3s kubectl -n argocd get applications            # root  Synced  Healthy
k3s kubectl -n wg-easy get endpointslice         # wg-easy-host  with the bridge address
```

A changed compose file recreates the wg-easy container: the tunnel drops for a few seconds. The
previous file is kept as `docker-compose.yml.<timestamp>~`.

## Backups

Nightly at 03:00 (before the reboot window) `node-infra-backup.timer` runs restic. It copies the
SQLite databases (k3s, wg-easy, Grafana) consistently with `sqlite3 .backup`, backs up the paths
listed in [`roles/backup/defaults/main.yml`](roles/backup/defaults/main.yml) encrypted to Backblaze
B2 and keeps 7 daily, 4 weekly and 6 monthly snapshots. On Sundays `node-infra-backup-check.timer`
verifies the repository and 5 % of the data. Everything else is rebuilt from git.

What is backed up is configured in `roles/backup/defaults/main.yml` (`backup_paths`,
`backup_exclude`, `backup_sqlite`); on the node it ends up in `/etc/node-infra/backup/`.

Set up once:

1. B2: a private bucket, lifecycle rule "Keep only the last version of the file", and an
   application key limited to that bucket with read and write access.
2. On the node: `/etc/node-infra/backup.env` from [`backup.env.example`](backup.env.example),
   mode 0600. Store `RESTIC_PASSWORD` in a password manager as well.
3. The next run enables the timers. The first backup creates the repository.

```bash
systemctl start node-infra-backup.service && journalctl -u node-infra-backup.service -n 30
systemctl list-timers 'node-infra-backup*'
set -a; . /etc/node-infra/backup.env; set +a
restic snapshots                                    # list
restic ls latest | head                             # look inside
restic restore latest --target /tmp/restore --include /etc/node-infra   # restore into a scratch dir
```

## Limits

- `--only-if-changed`: a run that fails is not retried until the next commit on `main`. To retry
  without a commit: `rm -rf /var/lib/node-infra/checkout && systemctl start node-infra-pull.service`.
- Whoever can change `main` gets root on the node. Protect `main` (ruleset: pull request required, no
  force push, no bypass for bots).
- Stop applying changes: `systemctl disable --now node-infra-pull.timer`.
- Security updates install daily; when one needs a reboot, the node reboots by itself at 04:00 server
  time and everything on it is down for a minute or two.
- SSH accepts keys only. Check `ssh -o PasswordAuthentication=no root@<node> true` works before merging
  changes to `roles/base`.

## Updates

[Renovate](https://github.com/renovatebot/renovate) (Mend Renovate App) opens a pull request for every
new version of what this repository pins: k3s, the wg-easy image
and the GitHub Actions. Merging it is the release approval; the node applies it like any other change.
Versions in role defaults carry a `# renovate:` comment that tells Renovate where to look
(`renovate.json`). The Dependency Dashboard issue lists all pending updates.

## Local checks

```bash
ansible-playbook --syntax-check site.yml
ansible-lint
```

CI runs the same two commands on every pull request.
