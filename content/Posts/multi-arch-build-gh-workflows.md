---
title: "Multi-Arch Build with GitHub Workflows"
date: 2026-05-20
author: Yadav Lamichhane
description: "Setup GitHub Actions workflow for building and pushing multi-architecture Docker images"
tags:
  - gh
---

<img width="1097" height="624" alt="Screenshot 2026-05-20 at 09 46 17" src="https://github.com/user-attachments/assets/3936e0e8-a440-4cfd-b562-6c39b11ef79c" />

Our platform has background jobs that runs across a mix of architectures — AMD64 for traditional cloud instances and ARM64 for AWS Graviton-based nodes. Rather than maintaining separate pipelines or Dockerfiles per architecture, we built a single GitHub Actions workflow that automatically builds, pushes, and merges multi-platform Docker images into one unified manifest.

When you push a Docker image tagged `my-image:latest`, that image is built for one specific CPU architecture. If your AMD64-built image lands on an ARM64 node (like a Graviton EC2 or an Apple Silicon), it either fails to run or runs through emulation — which is slow and unreliable in production.

The clean solution is a **multi-arch manifest** — a single image tag that points to the right architecture-specific image depending on where it's being pulled.

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

We have multiple jobs, each with its own Dockerfile under the `jobs/` directory. The pipeline:

1. **Discovers** all jobs dynamically
2. **Builds** each job for both `amd64` and `arm64` in parallel — on native runners
3. **Pushes** arch-specific tags to AWS ECR
4. **Merges** them into a single multi-platform manifest


## Dynamically Discover Jobs

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

## Merge into a Single Multi-Platform Manifest

Once both arch builds are pushed, we merge them into a single tag using `docker buildx imagetools`:

```bash
docker buildx imagetools create \
  -t ${ECR_PREFIX}/${job}:${IMAGE_TAG} \
  ${ECR_PREFIX}/${job}:${IMAGE_TAG}-amd64 \
  ${ECR_PREFIX}/${job}:${IMAGE_TAG}-arm64
```

Now when Kubernetes (or any container runtime) pulls `<ecr>/jobs/job-a:abc1234`, Docker automatically selects the right image for the node's architecture. No manual intervention. No separate tags in your Helm charts or manifests.


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

Multi-arch builds don't have to be complicated. With GitHub's native ARM runners and `docker buildx imagetools`, you get fast, clean, production-grade images for both AMD64 and ARM64 — with a single tag your infrastructure can rely on.
