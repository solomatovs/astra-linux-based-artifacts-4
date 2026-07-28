# Source artifacts (part 4)

This repository is a **git-only transport for build artifacts**: everything must be
retrievable with `git clone` alone (no GitHub Releases / LFS, which are served from
separate hosts that the build environment cannot reach).

It is the fourth artifact repository, split off from
[`astra-linux-based-artifacts-3`](https://github.com/solomatovs/astra-linux-based-artifacts-3.git)
to keep each repository at a workable size. It carries the sources for:

| component | contents                              |
|-----------|---------------------------------------|
| `rust`    | rustc 1.91.0–1.95.0 sources, ninja 1.11.1 |
| `uv`      | uv 0.9.30, 0.10.12, 0.11.19, 0.11.33 sources with vendored crates |

Both components continue chains that start in the earlier repositories and are
**not duplicated here**:

- `rust` — mrustc 0.12.0 and the rustc 1.90.0 sources it bootstraps live in
  [`astra-linux-based-artifacts`](https://github.com/solomatovs/astra-linux-based-artifacts.git).
  Building 1.91+ needs the `dmp/rust:1.90.0` image (or repo 1's artifacts to
  produce it) first; from there each version is built by the previous one.
- `uv` — 0.9.26 lives in
  [`astra-linux-based-artifacts-3`](https://github.com/solomatovs/astra-linux-based-artifacts-3.git).

The consuming `Makefile`/`Dockerfile` for each component lives in
[`astra-linux-based`](https://github.com/solomatovs/astra-linux-based.git), where the
artifact directory is git-ignored — only these sources need git transport.

All artifacts are committed **into git**. To respect GitHub's hard
**100 MB-per-file** limit, files larger than that are split into
`<file>.part-000`, `<file>.part-001`, … and only the pieces are committed.
[`artifacts-manifest.tsv`](artifacts-manifest.tsv) lists every artifact with its
path, size, sha256, and part count (`parts=0` means stored whole).

## On the build machine

```bash
git clone https://github.com/solomatovs/astra-linux-based-artifacts-4.git
cd astra-linux-based-artifacts-4
./scripts/assemble-artifacts.sh   # rebuild split files + verify all sha256
```

Whole files are already in place after clone; the script only reassembles the
split ones and checks integrity. Safe to re-run.

The artifacts are consumed from `astra-linux-based` via the `ARTIFACTS` variable,
which points at an out-of-tree directory (`../../ci-artifacts/<component>`), so
this tree maps onto it one-to-one — e.g. `rust/src/` here is
`ci-artifacts/rust/src/` there.

## Publishing (maintainer, needs push access)

Large files make the total push exceed GitHub's ~2 GB single-push limit, so the
push is done in size-bounded batches:

```bash
./scripts/split-artifacts.sh      # (re)generate .part-* pieces + manifest for files >95 MB
./scripts/push-artifacts.sh       # commit + push in <1.2 GB batches
```

## Adding a new artifact

1. Drop the file at its path under the relevant `<component>/` directory.
2. Run `./scripts/split-artifacts.sh` (splits it if >95 MB and refreshes the manifest).
3. Run `./scripts/push-artifacts.sh`.

Override `BATCH_MB` (default 1200) to change push batch size. As in the second and
third repositories, `split-artifacts.sh` selects files by size rather than by
extension, so any archive type is covered.
