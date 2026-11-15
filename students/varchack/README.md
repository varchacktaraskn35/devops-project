# DevOps Project: Node.js + Docker + Terraform + GitHub Actions

A small Node.js HTTP service that is containerized with Docker, published to Docker Hub
and deployed to an AWS EC2 instance (created by Terraform) by a GitHub Actions pipeline.

## Purpose

Show a complete DevOps flow: code → image → registry → cloud server, fully automated
after a `git push` to `main`.

## Repository layout

```
.github/workflows/ci.yml        CI/CD pipeline
students/varchack/
  app/                          Node.js app, Dockerfile, .dockerignore
  terraform/                    EC2 + Security Group as code
  README.md
```

## Run locally in Docker

```bash
cd students/varchack/app
docker build -t devops-project:local .
docker run --rm -p 3000:3000 devops-project:local
curl http://localhost:3000          # Hello from DevOps project! version=dev
curl http://localhost:3000/health   # {"status":"ok"}
```

## Create the infrastructure (Terraform)

Prerequisites: Terraform, AWS CLI configured (`aws configure`), an existing EC2 key pair
(its name goes into `terraform.tfvars` as `key_name`).

```bash
cd students/varchack/terraform
terraform init
terraform plan
terraform apply        # creates Security Group (22, 3000) and an Ubuntu 20.04 EC2
```

The output `instance_ip` is the public IP. Docker is installed on first boot via `user_data`.
Remove everything when done: `terraform destroy`.

## How the pipeline works

Triggered on every push to `main`:

1. **Checkout** the code.
2. **Tag**: the image tag is the first 7 characters of the commit hash.
3. **Docker Hub login** with `DOCKER_USERNAME` / `DOCKER_TOKEN`.
4. **Build & push** `DOCKER_USERNAME/devops-project:<hash>` from `students/varchack/app`.
5. **Find the EC2** created by Terraform (tag `Name=devops-project`) with the AWS keys.
6. **Decode the SSH key** from `EC2_SSH_KEY_B64`.
7. **Deploy**: over SSH remove the old `nodeapp` container, pull the new image, run it on port 3000.
8. **Smoke test**: `GET /health` must return 200.

## Configuration

GitHub Secrets (Settings → Secrets and variables → Actions):

| Secret | Purpose |
|---|---|
| `DOCKER_USERNAME` | Docker Hub login |
| `DOCKER_TOKEN` | Docker Hub access token (Read & Write) |
| `AWS_ACCESS_KEY_ID` | AWS key used by the pipeline to find the EC2 |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `EC2_SSH_KEY_B64` | Private `.pem` key, base64 in one line |

Environment variables of the app:

| Variable | Default | Meaning |
|---|---|---|
| `PORT` | `3000` | Listening port |
| `APP_VERSION` | `dev` | Version shown in the response (set to the commit hash on deploy) |

Terraform variables (`terraform.tfvars`): `aws_region`, `ami_id`, `instance_type`, `key_name`, `app_port`.

## Security notes

`.gitignore` excludes `*.pem`, `.env`, `.terraform/` and `terraform.tfstate*`; secrets live only in GitHub Secrets.
