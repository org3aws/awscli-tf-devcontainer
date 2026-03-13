# AWS CLI, Helper CLI + Terraform Minimal Dev Container

Minimal reusable dev container for AWS CLI, Terraform, and an included `aws-ec2` helper CLI, safe to share on GitHub.

## What is included
- AWS CLI v2
- Terraform (version pinned in `.devcontainer/devcontainer.json`)
- Ubuntu base image pinned to `ubuntu-24.04`
- VS Code extensions:
  - HashiCorp Terraform
  - AWS Toolkit

## Versioning policy
- Base image is pinned to an Ubuntu release tag for stable builds.
- Terraform is pinned via `TERRAFORM_VERSION`.
- AWS CLI is currently installed as latest v2 at build time.
- For full reproducibility, pin Docker base image by digest and pin AWS CLI installer version URL.

## Security and credential masking
This repository is set up so secrets stay local:
- `.gitignore` excludes `.aws/`, `.env`, `*.tfvars`, and Terraform state files.
- The dev container mounts your host `~/.aws` directory into the container at runtime.
- No credentials are stored in this repository.
- Redacted samples are provided in `examples/aws/`.

## Usage
1. Open this folder in VS Code.
2. Reopen in container (`Dev Containers: Reopen in Container`).
3. Ensure your local machine already has AWS credentials configured:
   - `~/.aws/credentials`
   - `~/.aws/config`
4. Inside container, verify:
   - `aws sts get-caller-identity`
   - `terraform version`

## aws-ec2 helper CLI
This dev container includes `aws-ec2`, a small EC2 fleet inspection helper installed at `/usr/local/bin/aws-ec2`.

Prerequisites:
- Valid AWS credentials and config available in `~/.aws`
- IAM permissions to call `ec2:DescribeRegions`, `ec2:DescribeInstances`, and `ec2:DescribeImages`

Basic usage:
- `aws-ec2 help`
- `aws-ec2 list`
- `aws-ec2 audit`
- `aws-ec2 images`

Commands:
- `aws-ec2 list`
   - Lists instances across all enabled regions
   - Shows: `Region`, `InstanceId`, `Name`, `Type`, `State`
- `aws-ec2 audit`
   - Audits instances across all enabled regions
   - Shows: `Region`, `InstanceId`, `Name`, `Type`, `State`, `Uptime` (days since launch)
- `aws-ec2 images`
   - Lists AMIs currently used by discovered instances
   - Shows: `Region`, `AMI ID`, `AMI Name`

Example workflow:
1. Confirm identity: `aws sts get-caller-identity`
2. List running and stopped instances: `aws-ec2 list`
3. Review uptime for aging hosts: `aws-ec2 audit`
4. Inspect AMIs in use: `aws-ec2 images`

## Sharing on GitHub
Before push, verify no sensitive data is tracked:
- `git status`
- `git ls-files | rg "(\.env|\.aws|tfstate|tfvars)"`
- `rg -n "(AKIA[0-9A-Z]{16}|aws_secret_access_key|aws_session_token)" .`

If needed, rotate credentials immediately if any were previously committed.

## CI/CD image publishing
This repository includes a GitHub Actions workflow that builds the dev container image from `.devcontainer/Dockerfile` and publishes it to GitHub Container Registry (`ghcr.io`).

Behavior:
- Pull requests build the image only, which catches Dockerfile regressions before merge.
- Pushes to `main` publish the image tagged as `latest` and with the commit SHA.
- Version tags like `v1.2.3` publish matching image tags.
- `workflow_dispatch` lets you trigger a manual publish.

Published image name:
- `ghcr.io/<owner>/<repo>`

Repository requirements:
- GitHub Actions must be enabled.
- The workflow uses the built-in `GITHUB_TOKEN` with `packages: write` permission.
- If your default branch is not `main`, update the workflow trigger accordingly.

After the first publish, you can pull the image with:
- `docker pull ghcr.io/<owner>/<repo>:latest`
