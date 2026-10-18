# node-infra

Host setup of the k3s node, applied with Ansible. Everything below Kubernetes lives here.

Only `main` is applied. A merged pull request is the release approval: a systemd timer on the node runs
`ansible-pull` against `main` and applies `site.yml` when there is a new commit. Nothing outside the node
needs access to it.

This repository is public. No addresses, hostnames or secrets go into it; per-node values live in
`/etc/node-infra/pull.env` on the node.

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
