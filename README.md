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

Today the playbook installs only the `ansible-pull` timer, so that every later change arrives by merge.
Planned roles, each in its own pull request: `base` (packages, updates), `k3s`, `wg-easy`, `dnsmasq`,
`argocd-bootstrap`.

## Layout

```text
ansible.cfg
inventory.yml          localhost only (ansible-pull runs on the node)
site.yml               entry point for ansible-pull
roles/
  ansible_pull/        installs ansible-core and git, the service and the timer
```

## Environment on the node

`/etc/node-infra/pull.env`, written by hand, read by `node-infra-pull.service`:

| Variable | Meaning |
| --- | --- |
| `NODE_INFRA_REPO_URL` | Repository to pull, e.g. `https://github.com/gpt-partners/node-infra.git` |
| `NODE_INFRA_BRANCH` | Branch to apply, `main` |

The playbook stops with a clear error if the file is missing.

## First run (once, by hand, as root on the node)

```bash
apt-get update && apt-get install -y ansible-core git

install -d -m 0755 /etc/node-infra
cat > /etc/node-infra/pull.env <<'EOF'
NODE_INFRA_REPO_URL=https://github.com/gpt-partners/node-infra.git
NODE_INFRA_BRANCH=main
EOF
chmod 0644 /etc/node-infra/pull.env

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
```

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
