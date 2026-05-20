---
title: "Multi-Arch Build with GitHub Workflows"
date: 2025-12-27
author: Yadav Lamichhane
description: "Setup GitHub Actions workflow for building and pushing multi-architecture Docker images"
tags:
  - gh
---

# Building Multi-Arch Docker Images for ARM and AMD64 with GitHub Actions and AWS ECR

Our AI platform runs background jobs across a mix of architectures — AMD64 for traditional cloud instances and ARM64 for AWS Graviton-based nodes. Rather than maintaining separate pipelines or Dockerfiles per architecture, we built a single GitHub Actions workflow that automatically builds, pushes, and merges multi-platform Docker images into one unified manifest.

Here's exactly how we did it.

---

## The Problem with Single-Arch Images

When you push a Docker image tagged `my-image:latest`, that image is built for one specific CPU architecture. If your AMD64-built image lands on an ARM64 node (like a Graviton EC2 or an Apple Silicon dev machine), it either fails to run or runs through emulation — which is slow and unreliable in production.

The clean solution is a **multi-arch manifest** — a single image tag that points to the right architecture-specific image depending on where it's being pulled.

---

## Our Setup at a Glance

```
jobs/
├── job-a/
│   └── Dockerfile
├── job-b/
│   └── Dockerfile
└── job-c/
    └── Dockerfile
```

We have multiple background jobs, each with its own Dockerfile under the `jobs/` directory. The pipeline:

1. **Discovers** all jobs dynamically
2. **Builds** each job for both `amd64` and `arm64` in parallel — on native runners
3. **Pushes** arch-specific tags to AWS ECR
4. **Merges** them into a single multi-platform manifest

---

## Step 1: Dynamically Discover Jobs

Instead of hardcoding job names, we auto-discover any directory under `jobs/` that contains a Dockerfile:

```yaml
- id: set-matrix
  run: |
    JOBS=$(find jobs -mindepth 1 -maxdepth 1 -type d \
      -exec test -e "{}/Dockerfile" ';' -print \
      | jq -R -s -c 'split("\n")[:-1]')
    echo "matrix=$JOBS" >> $GITHUB_OUTPUT
```

This outputs a JSON array like `["jobs/job-a", "jobs/job-b"]` which feeds directly into the build matrix. Adding a new job is as simple as creating a new folder — no pipeline changes needed.

---

## Step 2: Environment-Aware Configuration

The pipeline supports multiple deployment targets — production, staging, and dev environments — using Git tags and branches to determine where to push:

```yaml
# Push to production ECR on ci/prod tag
if: ${{ github.ref == 'refs/tags/ci/prod' }}

# Push to staging ECR on everything else
if: ${{ github.ref != 'refs/tags/ci/prod' }}
```

Image tags are derived from context:
- `master` branch → tagged as `master`
- `ci/staging` tag → tagged as `staging`
- `ci/prod` tag → tagged as the short Git SHA (e.g. `a3f92c1`)

This gives full traceability in production while keeping dev/staging tags human-readable.

---

## Step 3: Build on Native Runners — Not Emulation

This is the most important architectural decision. Many guides use QEMU emulation to build ARM images on AMD64 runners. It works, but it's significantly slower — sometimes 5–10x — for compute-heavy builds.

Instead, we use **native runners for each architecture**:

```yaml
matrix:
  config:
    - platform: linux/amd64
      runner: ubuntu-latest       # Standard GitHub AMD64 runner
      arch: amd64
    - platform: linux/arm64
      runner: ubuntu-24.04-arm    # GitHub's native ARM64 runner
      arch: arm64
```

Each job in the matrix runs on hardware that matches its target platform. The build is fast, native, and deterministic.

The build step pushes arch-specific tags:

```bash
docker buildx build \
  --platform linux/arm64 \
  -t <ecr>/<job>:<tag>-arm64 \
  --push \
  -f jobs/job-a/Dockerfile \
  .
```

This results in two images per job per tag:
- `<ecr>/jobs/job-a:abc1234-amd64`
- `<ecr>/jobs/job-a:abc1234-arm64`

---

## Step 4: Merge into a Single Multi-Platform Manifest

Once both arch builds are pushed, we merge them into a single tag using `docker buildx imagetools`:

```bash
docker buildx imagetools create \
  -t ${ECR_PREFIX}/${job}:${IMAGE_TAG} \
  ${ECR_PREFIX}/${job}:${IMAGE_TAG}-amd64 \
  ${ECR_PREFIX}/${job}:${IMAGE_TAG}-arm64
```

Now when Kubernetes (or any container runtime) pulls `<ecr>/jobs/job-a:abc1234`, Docker automatically selects the right image for the node's architecture. No manual intervention. No separate tags in your Helm charts or manifests.

---

## The Full Pipeline Flow

```
Push to master / tag
        │
        ▼
┌─────────────────┐
│  Discover Jobs  │  → finds all Dockerfiles dynamically
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Setup Config   │  → resolves ECR target, image tag, IAM role
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│           Build Matrix (parallel)         │
│                                          │
│  job-a / amd64   job-a / arm64           │
│  job-b / amd64   job-b / arm64      ...  │
│                                          │
│  (native runners — no emulation)         │
└──────────────────────┬───────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────┐
│         Merge Manifests (per job)         │
│                                          │
│  amd64 image + arm64 image               │
│         → single multi-arch tag          │
└──────────────────────────────────────────┘
```

---

## AWS ECR + OIDC Authentication

We use GitHub's OIDC provider to assume an AWS IAM role — no long-lived credentials stored in secrets:

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-region: us-east-1
    role-to-assume: ${{ needs.setup.outputs.role_arn }}
```

Different IAM roles are used for staging vs production, scoped with least-privilege ECR push permissions. This is significantly more secure than storing `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as GitHub secrets.

---

## Key Takeaways

| Decision | Why |
|---|---|
| Native ARM runners instead of QEMU | 5–10x faster builds, no emulation quirks |
| Dynamic job discovery via `find` | Zero pipeline changes when adding new jobs |
| Arch-specific tags + manifest merge | Clean single tag for Kubernetes to consume |
| OIDC instead of static credentials | No long-lived secrets, per-environment roles |
| Git tag–based environment routing | Full traceability, clear promotion model |

---

## What's Next

A few improvements we're exploring:

- **Layer caching** — using ECR as a BuildKit cache backend to speed up repeated builds
- **Build provenance** — attaching SLSA attestations to images for supply chain security
- **Selective builds** — only rebuilding jobs whose source files changed, using path-based change detection

---

Multi-arch builds don't have to be complicated. With GitHub's native ARM runners and `docker buildx imagetools`, you get fast, clean, production-grade images for both AMD64 and ARM64 — with a single tag your infrastructure can rely on.
