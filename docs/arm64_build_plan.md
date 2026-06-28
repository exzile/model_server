# ARM64 (aarch64) Docker build — implementation plan

Tracking: issue [#2648](https://github.com/openvinotoolkit/model_server/issues/2648).
Supersedes the stale proof-of-concept draft PR
[#2485](https://github.com/openvinotoolkit/model_server/pull/2485) (last commit
2024-06, OpenVINO 2024.1).

## Goal & scope

Produce a **from-source, buildable minimal ARM64 OVMS image** off current `main`,
without regressing the existing x86_64 build. Out of scope (separate efforts):
official multi-arch image publishing (owned by Intel release/CI), and full feature
parity on ARM (Mediapipe graphs, Azure blob, Python binding).

Initial ARM image is minimal, matching the only configuration known to build:
`MEDIAPIPE_DISABLE=1 PYTHON_DISABLE=1 TOKENIZERS=0` and S3 disabled.

## Why not just rebase #2485

#2485 makes the build pass by **deleting** ~500 lines of `S3FileSystem` and
hard-swapping every `x86_64` lib path to `aarch64`. Merging that would break the
x86_64 build. The correct pattern is **architecture-conditional** selection
(Bazel `select()` on `@platforms//cpu`, Dockerfile `ARG TARGETARCH`), leaving the
x86 path untouched.

## Hardcoded-x86 inventory on current `main` (verified)

| File | Location | Issue | Fix approach |
|------|----------|-------|--------------|
| `Dockerfile.ubuntu` | L151-155 | Bazel installer pinned `linux-x86_64.sh` | Select installer/binary by `TARGETARCH` (arm64 → `bazel-$V-linux-arm64`) |
| `Dockerfile.ubuntu` | L203, 264, 222, 233 | OpenVINO lib dir `runtime/lib/intel64` | `intel64` is the x86 dir; aarch64 package uses `runtime/lib/aarch64` — parameterize via `ARG OV_LIBDIR` |
| `Dockerfile.ubuntu` | L369-370 | bazel-out glob `k8-*` (x86 cpu) | aarch64 emits `aarch64-*`; glob both or derive from `TARGETARCH` |
| `Dockerfile.ubuntu` | L189-190 | `DLDT_PACKAGE_URL` | pass aarch64 OpenVINO package URL at build time (build-arg, no code change) |
| `Makefile` | L381 etc. | `docker build` has no `--platform` | add `--platform linux/$(TARGETARCH)` driven by a new make var, default `amd64` |
| `WORKSPACE` | L131-132 | `curl` hdrs/srcs `x86_64-linux-gnu` | `select()` aarch64 vs x86_64 paths |
| `BUILD.bazel` / `src/BUILD` | aws-sdk-cpp deps | S3/AWS SDK on ubuntu/redhat | gate AWS SDK + `S3FileSystem` behind a `disable_s3` config setting instead of deleting code |
| `src/filesystemfactory.cpp`, `src/s3filesystem.*` | — | S3 impl | compile-guard with a macro (e.g. `OVMS_DISABLE_S3`) — do **not** delete |

## Step sequence

1. **Build scaffolding (no code regression):** add `TARGETARCH` plumbing to
   `Makefile` + `Dockerfile.ubuntu` (bazel installer, OV libdir, bazel-out glob,
   `--platform`). x86 defaults unchanged. ← _safe, mechanical_
2. **WORKSPACE `select()`** for curl (and any other x86-pinned `new_local_repository`).
3. **S3 compile-guard:** introduce `OVMS_DISABLE_S3` macro + Bazel `config_setting`;
   wrap `S3FileSystem` use in `filesystemfactory.cpp`. Keep full S3 source intact.
4. **Local ARM build validation** on aarch64 hardware (the x86 box cannot build this):
   `OVMS_CPP_DOCKER_IMAGE=arm BASE_IMAGE=arm64v8/ubuntu:24.04 OV_USE_BINARY=1 \
    DLDT_PACKAGE_URL=<aarch64 OV pkg> TOKENIZERS=0 BASE_OS=ubuntu24 \
    MEDIAPIPE_DISABLE=1 PYTHON_DISABLE=1 RUN_TESTS=0 make docker_build`
5. **Smoke test:** serve a small CPU model, run a gRPC/REST inference on ARM.
6. **Docs:** add an "Experimental ARM64 build" section; open PR superseding #2485.

## Open coordination questions (for maintainers before PR)

- Does internal CVS-129299 supersede this, or is a community PR welcome?
- Target OpenVINO aarch64 package version to pin for `DLDT_PACKAGE_URL`?
- Preferred base OS for ARM (ubuntu24 vs ubuntu22)?

## Validation constraint

ARM build/runtime validation must happen on aarch64 hardware. The current dev box
(Windows/x86) can author and review changes but cannot compile or run the ARM image.
