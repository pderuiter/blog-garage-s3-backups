# blog-garage-s3-backups

Companion code for the blog post
[Setting up Garage S3 Storage with Docker for Kubernetes Backups](https://webuildyourcloud.com/setting-up-garage-s3-storage-with-docker-for-kubernetes-backups/).

It builds the setup from the post on two Ubuntu 24.04 VMs:

- **Backup server (Part 1).** Docker CE, then Garage v1.1.0 from the post's `garage.toml` and `docker-compose.yml`,
  with Traefik labels for TLS. The role sets up a single-node layout (`dc1`, 1T), the `backup-user` key, and the
  `gitlab-backup`/`keycloak-backup`/`sonarqube-backup` buckets with read/write/owner grants and a 15-day lifecycle
  expiration.
- **Kubernetes (Part 2).** A single-node k3s cluster with the post's backup CronJob pattern. An Alpine init container
  bundles `wget`, the musl libraries and the CA certificates for the ntfy alert, a `postgres:16.1` container runs
  `pg_dump | gzip` under `set -eo pipefail`, and `trap notify_failure EXIT` sends the ntfy alert.

## Differences from the post

The post uploads with the MinIO Client (`mc`). The open-source `mc` is archived: its download URL now returns
`410 Gone`, and the `minio/mc` image is gone from Docker Hub. This repo uses the **aws CLI** instead:

- **CronJob.** A second init container copies the self-contained aws CLI v2 from `amazon/aws-cli:2.41.0` into the
  job. The backup runs `aws s3api head-bucket` and `aws s3 cp`, with
  `AWS_REQUEST_CHECKSUM_CALCULATION=when_required` and `AWS_RESPONSE_CHECKSUM_VALIDATION=when_required`. Without
  them, Garage v1.x rejects the new default checksums with `invalid checksum algorithm`.
- **What still doesn't work.** The post's other aws CLI problem remains: `DeleteObject` fails on Garage v1.x. The
  backup never deletes objects; expiry is handled by the lifecycle rules.
- **Lifecycle rules.** These are set with `aws s3api put-bucket-lifecycle-configuration`, the workaround the post
  gives for Garage v1.x, where `mc ilm rule add` fails.

Credentials come from plain Kubernetes Secrets by default. Set `garage_backup_secret_source: external-secrets` to get
the post's ExternalSecrets instead (External Secrets Operator and a `vault-backend` ClusterSecretStore must exist,
with the values stored in Vault as the post shows).

## What's in here

| Path | What it does |
| --- | --- |
| `terraform/` | Two Proxmox VE VMs from the Ubuntu 24.04 cloud image (bpg/proxmox provider): `garage` and `k8s`. Outputs an Ansible inventory. |
| `ansible/roles/docker_engine` | Docker CE, containerd and the Compose v2 plugin. |
| `ansible/roles/garage` | Steps 1-7 of Part 1. The `admin_token` file is written with mode 0600 (Garage refuses to start if it is world-readable); `garage.toml` and `docker-compose.yml` follow the post. The role creates the Traefik network if missing, applies the layout once, creates the key and buckets, and sets the lifecycle rules. |
| `ansible/roles/k3s` | k3s v1.37.2+k3s1, Helm v3.19.0, and a virtualenv with the kubernetes Python client. |
| `ansible/roles/garage_backup` | Part 2: a namespace per job, the `garage-credentials` and `<app>-db-password` Secrets (or ExternalSecrets), and the CronJob. |
| `ansible/site.yml` | Garage, then k3s, then the CronJobs. |
| `tests/demo-db.yml` | A throw-away `postgres:16.1` as `gitlab-postgres.gitlab.svc` with 500 rows, for testing without a real GitLab. |
| `tests/verify.yml` | `kubectl create job --from=cronjob/gitlab-backup`, waits for it, checks the log and lists the dump in Garage. |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VMs).
- Ansible >= 2.16 (ansible-core) and the collections in `ansible/requirements.yml`.
- SSH access with sudo to both machines. The Terraform cloud-init user is `ubuntu`.
- Traefik on the backup server with a certificate resolver (see `blog-traefik-letsencrypt-docker`), and DNS for
  `backups.example.com`. Without Traefik, set `garage_publish_ports` and point `garage_backup_url` at the published
  port.
- An ntfy topic for the failure alerts.

## Quick start

```bash
# 1. Machines (optional; any two Ubuntu 24.04 hosts work)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in endpoint, token, node, IPs, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configuration
cd ../ansible
ansible-galaxy collection install -r requirements.yml
$EDITOR group_vars/all/vars.yml                  # garage_host, buckets, jobs, ntfy URL
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml   # rpc_secret, admin token (openssl rand -hex 32), DB password
ansible-playbook site.yml --ask-vault-pass

# 3. Optional end-to-end check
ansible-playbook ../tests/demo-db.yml --ask-vault-pass   # only without a real database
ansible-playbook ../tests/verify.yml --ask-vault-pass
```

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `garage_image` | `dxflrs/garage:v1.1.0` | As in the post. v2.x fixes `mc ilm`, but its CLI differs slightly (`key info --show-secret` is the same) |
| `garage_rpc_secret` / `garage_admin_token` | from the vault | 64 hex characters each |
| `garage_host` | `backups.example.com` | Traefik host rule; also the base of `root_domain` |
| `garage_data_dir` | `/data-pool/backups` | Data on the host (a dedicated disk or ZFS dataset) |
| `garage_traefik_network` / `garage_traefik_certresolver` | `traefik_network` / `production` | Traefik network and resolver |
| `garage_publish_ports` | `[]` | Extra published ports, e.g. `["127.0.0.1:3900:3900"]` |
| `garage_layout_zone` / `garage_layout_capacity` | `dc1` / `1T` | Single-node layout |
| `garage_key_name` | `backup-user` | API key for the backups |
| `garage_buckets` | the post's three buckets, 15 days | `{name, expire_days}` |
| `garage_backup_url` | `https://{{ garage_host }}` | Endpoint the CronJobs upload to |
| `garage_backup_ntfy_url` | `https://ntfy.example.com/backups` | ntfy topic for failures |
| `garage_backup_jobs` | `gitlab-backup` at 03:00 | `{name, schedule, bucket, db_host, db_user, db_name, db_password, namespace}` |
| `garage_backup_secret_source` | `kubernetes` | `kubernetes` (plain Secrets) or `external-secrets` (the post's ExternalSecrets) |
| `garage_backup_aws_cli_image` | `amazon/aws-cli:2.41.0` | Where the aws CLI comes from |

## Restoring

The post has no restore step. This works (tested):

```bash
aws --endpoint-url https://backups.example.com --region garage \
  s3 cp s3://gitlab-backup/db/<file>.sql.gz - | gunzip | psql -v ON_ERROR_STOP=1 -h <host> -U gitlab -d <empty database>
```

## Tested

`site.yml` was run against fresh Ubuntu 24.04 systemd containers (Docker 29, k3s v1.37.2+k3s1). The second run
reported `changed=0`. Garage was reached directly over HTTP (`garage_publish_ports`); no Traefik in front.
`tests/demo-db.yml` and `tests/verify.yml` passed:
- The job logged `Backup completed successfully`, and the dump was in `gitlab-backup/db/`.
- Restoring it into an empty database gave all 500 rows back.
- A failing run (the aws CLI not found, exit 127) sent the ntfy alert: `Backup Failed: gitlab-backup`, priority high,
  tag `rotating_light`.

The `external-secrets` mode was not run. ansible-lint (production profile), yamllint, `terraform validate` and tflint
are clean. The Terraform code has not been applied against a real Proxmox cluster.

## License

MIT
