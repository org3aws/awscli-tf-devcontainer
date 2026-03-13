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

### AWS credentials and profile (`.aws` + `AWS_PROFILE`)
The container bind-mounts your host `~/.aws` into the container user's home (`$HOME/.aws`), so configure credentials on the host first.

If you do not already have local AWS config files, use the redacted samples:

```bash
mkdir -p ~/.aws
cp examples/aws/credentials.example ~/.aws/credentials
cp examples/aws/config.example ~/.aws/config
chmod 600 ~/.aws/credentials ~/.aws/config
```

Set the profile you want AWS CLI and Terraform to use:

```bash
export AWS_PROFILE=default
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
```

Inside the dev container, you can confirm or override the profile for the current shell:

```bash
echo "$AWS_PROFILE"
export AWS_PROFILE=default
aws sts get-caller-identity
```

To persist in your shell startup file:

```bash
echo 'export AWS_PROFILE=default' >> ~/.bashrc
```

### Running directly with Docker (no devcontainer)
If you run the image directly, mount your host `~/.aws` to the container user's home directory (region comes from `~/.aws/config`).

```bash
CONTAINER_HOME=$(docker run --rm ghcr.io/org3aws/awscli-tf-devcontainer:latest sh -lc 'printf %s "$HOME"')
```

```bash
docker run --rm -it \
   -v "$HOME/.aws:${CONTAINER_HOME}/.aws:ro" \
   -e AWS_PROFILE=default \
   ghcr.io/org3aws/awscli-tf-devcontainer:latest \
   aws sts get-caller-identity
```

If you need an interactive shell:

```bash
docker run --rm -it \
   -v "$HOME/.aws:${CONTAINER_HOME}/.aws:ro" \
   -e AWS_PROFILE=default \
   ghcr.io/org3aws/awscli-tf-devcontainer:latest \
   bash
```

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

### Use this repository as a shared remote
Yes, this repository can be used as a normal Git remote for your team.

Typical team usage:

```bash
git clone https://github.com/org3aws/awscli-tf-devcontainer.git
cd <repo>
```

Then open in VS Code and run `Dev Containers: Reopen in Container`.

If you want this to be a starter project, you can also mark it as a GitHub template repository so others can create their own copy.

### Use published image as base for other devcontainers
After this repo publishes to GHCR, other repositories can consume the image directly instead of rebuilding tools each time.

Example `.devcontainer/devcontainer.json` in another repo:

```json
{
   "name": "my-project-dev",
   "image": "ghcr.io/org3aws/awscli-tf-devcontainer:latest",
   "mounts": [
      "source=${localEnv:HOME}/.aws,target=${containerEnv:HOME}/.aws,type=bind,consistency=cached"
   ],
   "remoteEnv": {
      "AWS_PROFILE": "default"
   }
}
```

If you want deterministic builds in downstream repos, pin a version tag (for example `ghcr.io/org3aws/awscli-tf-devcontainer:v1.2.3`) instead of `latest`.

## CI/CD image publishing
This repository includes a GitHub Actions workflow that builds the dev container image from `.devcontainer/Dockerfile` and publishes it to GitHub Container Registry (`ghcr.io`).

Behavior:
- Pull requests build the image only, which catches Dockerfile regressions before merge.
- Pushes to `main` publish the image tagged as `latest` and with the commit SHA.
- Version tags like `v1.2.3` publish matching image tags.
- `workflow_dispatch` lets you trigger a manual publish.

Published image name:
- `ghcr.io/org3aws/awscli-tf-devcontainer`

Repository requirements:
- GitHub Actions must be enabled.
- The workflow uses the built-in `GITHUB_TOKEN` with `packages: write` permission.
- If your default branch is not `main`, update the workflow trigger accordingly.

After the first publish, you can pull the image with:
- `docker pull ghcr.io/org3aws/awscli-tf-devcontainer:latest`
