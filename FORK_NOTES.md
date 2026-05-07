# Fork notes

This is `Hal0ai/amd-strix-halo-toolboxes`, a friendly fork of
[`kyuz0/amd-strix-halo-toolboxes`](https://github.com/kyuz0/amd-strix-halo-toolboxes).

## Why we forked

We need llama-server container images that ship with `llama-server` as
their `ENTRYPOINT` (rather than the toolbox `/bin/bash`), so that
[`Hal0ai/hal0`](https://github.com/Hal0ai/hal0)'s SlotManager can
manage them as systemd-driven services. The two new files

- `toolboxes/Dockerfile.vulkan-radv-server`
- `toolboxes/Dockerfile.rocm-7.2.2-server`

are byte-for-byte identical to upstream's canonical
`Dockerfile.vulkan-radv` and `Dockerfile.rocm-7.2.2` through both
build and runtime stages. The only changes are:

1. `curl` added to the runtime `microdnf install` line so the
   HEALTHCHECK can probe `/health`.
2. Replaced `CMD ["/bin/bash"]` with `EXPOSE 8080`, a HEALTHCHECK,
   `ENTRYPOINT ["llama-server"]`, and `CMD ["--help"]`.

Same patches were proposed upstream as
[kyuz0#86 (Vulkan)](https://github.com/kyuz0/amd-strix-halo-toolboxes/pull/86) and
[kyuz0#87 (ROCm)](https://github.com/kyuz0/amd-strix-halo-toolboxes/pull/87).
Those PRs remain open. We forked rather than wait so that haloai's
ADR-0008 Phase 2 (containerise the daily-driver inference slots)
isn't blocked on review timing.

## Re-converge plan

If/when the upstream PRs merge and kyuz0 publishes the corresponding
Docker Hub tags (`docker.io/kyuz0/amd-strix-halo-toolboxes:vulkan-radv-server`
and `:rocm-7.2.2-server`):

1. Smoke-test those images in a haloai dev slot.
2. Update `lib/providers/llama_server.py:_KYUZ0_IMAGES` (haloai side)
   to point at the upstream Docker Hub refs.
3. Keep this fork warm for ≈30 days as rollback safety.
4. After 30 days of stable upstream, archive this repo and remove the
   `upstream-sync` cron from the active config.

## Image registry

We publish to GHCR rather than Docker Hub so the divergence is
visually obvious and so we don't share rate-limit budget with kyuz0:

| Backend | Tag | Image ref |
| :--- | :--- | :--- |
| Vulkan (RADV) — service | `vulkan-radv-server` | `ghcr.io/hal0ai/amd-strix-halo-toolboxes:vulkan-radv-server` |
| ROCm 7.2.2 — service    | `rocm-7.2.2-server`  | `ghcr.io/hal0ai/amd-strix-halo-toolboxes:rocm-7.2.2-server`  |

Builds are driven by `.github/workflows/ghcr-publish.yml`. Rebuild
cadence is daily at 03:15 UTC plus on push when a `*-server`
Dockerfile changes.

## Upstream sync

`.github/workflows/upstream-sync.yml` runs daily at 06:00 UTC and
fast-forwards our `main` to `upstream/main` whenever possible. Since
our additions are new files (no edits to upstream's existing
Dockerfiles or workflows), FF is the typical path. The workflow opens
a PR for human review when FF isn't possible — usually a sign that
upstream changed an existing Dockerfile we should diff against our
`*-server` siblings.

The workflow needs a `SYNC_TOKEN` secret (fine-grained PAT). See the
workflow file for the exact scopes.

## License

At fork time (2026-05-07), the upstream repo did not contain a
`LICENSE` file. We forked under GitHub's standard fork terms (which
permit forking and downstream contributions for any public repo). Our
own additions in this fork — `Dockerfile.*-server`, `FORK_NOTES.md`,
the GHCR publish and upstream-sync workflows — are licensed
permissively (MIT) and may be used freely by anyone, including
upstream if/when these PRs land.

We've asked upstream to adopt an explicit license. Until that happens,
downstream consumers of the *built images* are in the same legal
posture as consumers of `docker.io/kyuz0/...:*` — kyuz0's published
intent (a public repo with an open PR queue) is the operative signal.

## Credits

All build-stage logic — Strix Halo `gfx1151` flags, the
`-mllvm --amdgpu-unroll-threshold-local=600` ROCm 7 perf-regression
workaround, `LLAMA_HIP_UMA=ON`, the Vulkan/RADV setup, the
`llama-grammar.patch` — is kyuz0's work. This fork only adds the
service-mode runtime tail. If you find the *-server images useful,
please [support kyuz0](https://buymeacoffee.com/dcapitella).
