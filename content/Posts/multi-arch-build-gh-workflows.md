---
title: "Multi-Arch Docker Build with GitHub Actions"
date: 2026-05-20
author: Yadav Lamichhane
description: "Setup GitHub Actions workflow for building and pushing multi-architecture Docker images"
tags:
  - gh
---

![Multi-Arch Docker Build](https://i.imgur.com/4vQXlq8.png)

If you've ever built a multi-arch Docker image using QEMU emulation on GitHub Actions and
watched the clock tick past 15 minutes, you already know the pain. In this post, I'll walk
through how switching to native runners cut our build times dramatically — and what the
workflow looks like in practice.

## The Problem with QEMU

QEMU is the go-to approach for building multi-arch images on a single runner. You set up the
emulator, point Buildx at it, and in one job you get both `linux/amd64` and `linux/arm64`
images. Simple — but slow.

The reason is fundamental: QEMU emulates the target CPU in software. Every ARM instruction
your build executes is translated on the fly by the x86 host. For a Go binary this might be
tolerable, but for anything with native dependencies — CGo, Python packages with C extensions,
or heavy layer operations — the overhead is brutal.

Here's what the single-job QEMU workflow snippets looks like:

```yaml
name: Build and Push Multi-Arch Docker Image

on:
  push:
  workflow_dispatch:

jobs:
  build-and-push:
    # ... rest of the steps
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Clean and minimal — but when your arm64 build is emulated on an amd64 runner, expect it to
run **3–5x slower** than a native build. A build that takes 3 minutes natively can easily
stretch to 12–15 minutes under QEMU.

Here's what a QEMU-emulated arm64 build looks like in practice — notice the build duration:

![QEMU Build Time](https://i.imgur.com/zFtIaZe.png)

## The Native Approach: Parallel Jobs + Manifest Merge

The fix is straightforward: build each architecture on its own native runner in parallel, then
merge the two images into a single multi-arch manifest.

```yaml
name: Build and Push Multi-Arch Docker Image

on:
  push:
  workflow_dispatch:

jobs:
  # Builds natively on an x86 runner
  build-amd64:
    # ... rest of the steps
      - name: Build and push amd64
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64
          push: true
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}-amd64
          cache-from: type=gha,scope=amd64
          cache-to: type=gha,mode=max,scope=amd64

  # Builds natively on an ARM runner — no emulation
  build-arm64:
     # ... rest of the steps
      - name: Build and push arm64
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/arm64
          push: true
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}-arm64
          cache-from: type=gha,scope=arm64
          cache-to: type=gha,mode=max,scope=arm64

  merge-manifests:
        # ... rest of the steps
      - name: Create and push multi-arch manifest
        run: |
          docker buildx imagetools create \
            -t ${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -t ${{ env.IMAGE_NAME }}:latest \
            ${{ env.IMAGE_NAME }}:${{ github.sha }}-amd64 \
            ${{ env.IMAGE_NAME }}:${{ github.sha }}-arm64
```

The flow looks like this:

```
build-amd64 (ubuntu-latest)       ──┐
                                    ├──► merge-manifests──► image:latest
build-arm64 (ubuntu-linux-arm64)  ──┘
```

And here's how it looks in the GitHub Actions UI — both architecture builds running in parallel:

![Parallel Native Builds](https://i.imgur.com/oR7QAzr.png)

Once both jobs complete, the manifest merge job ties them together.

## Build Time Comparison

| Approach | amd64 build | arm64 build |
|---|---|---|
| QEMU (single job) | ~43 sec | ~5 min (emulated) |
| Native (parallel jobs) | ~43 sec | ~48 sec (native) |

The arm64 build under QEMU is the bottleneck. Running it natively on an ARM runner eliminates
the emulation overhead entirely, and since both jobs run in parallel, the overall pipeline time
drops to just over the time of a single native build plus the manifest merge (under a minute).

## Setting Up a Native ARM Runner on GitHub

GitHub now provides hosted `linux/arm64` runners for public and private repositories on paid plans. To enable them:

1. Navigate to your organization **Settings** → **Actions** → **Runners**.
2. Click **New runner** → **New GitHub-hosted runner**.
3. Configure the runner:
   - **Name:** `ubuntu-linux-arm64` (or any label you prefer)
   - **Image:** Ubuntu latest
   - **Architecture:** `arm64`
   - **Size:** Choose based on your build needs (e.g., 4-core)
4. Save the runner group and grant access to the desired repositories.

Then reference the label in your workflow:

```yaml
build-arm64:
  runs-on: ubuntu-linux-arm64
  steps:
    - uses: actions/checkout@v4
    # ... rest of the steps
```

## When to Still Use QEMU

Native runners are not always available or free. QEMU still makes sense when your images are
small and build fast, your CI provider doesn't offer ARM runners, or you're prototyping and
want a simple single-job setup.

For production workloads or anything with a meaningful build time, native runners pay for
themselves quickly in developer time saved.

## Try It Yourself

The complete working example from this post is available on [GitHub](https://github.com/coderzplayground/multi-arch-build)
. Feel free to clone it, run the workflows, and experiment with your own images:

## Wrapping Up

The change is not dramatic in terms of workflow complexity — you're splitting one job into
two and adding a merge step. But the payoff in build time is significant. If your team is
pushing to main frequently or running builds on every PR, shaving 10+ minutes per run adds
up fast.

Give the native approach a try — your team will notice the difference.
