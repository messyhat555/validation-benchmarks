# Fork fixes — `validation-benchmarks`

This fork of [`arthurgervais/validation-benchmarks`](https://github.com/arthurgervais/validation-benchmarks)
repairs the benchmarks that no longer built and adds bulk-build tooling. **All 104 XBEN
challenges build.** The corpus is consumed by `benchmark-emile` as its XBEN baseline.

Delivered in two merged PRs:

- **#1** — Fix 12 broken benchmark builds; add `build-all`.
- **#2** — Normalize `benchmark.json` `level` to int.

## Prerequisite

Building requires the Docker **buildx** plugin so Docker Compose honours the `platform:`
field at build time. Without it, the platform-pinned base images fail to resolve.

## Build every benchmark

The root `Makefile` provides two extra targets (they build, they do **not** run):

```bash
make build-all     # build all 104, continue past failures, print a summary (non-zero exit if any failed)
make clean-all     # drop every benchmark's .xben_build_done guard so the next build re-runs
```

Individual benchmarks are unchanged: `make build BENCHMARK=XBEN-001-24`.

## Build fixes, by root cause

### 1. `mysql:5.7.15` base image no longer extractable — 8 benchmarks

**003, 043, 063, 066, 081, 088, 095, 104**

The 2016-era `mysql:5.7.15` image fails to unpack under modern containerd
(`failed to extract layer … content digest not found`); it reproduces on a fresh pull,
so it is the image, not a corrupt cache.

**Fix:** repin each `mysql/Dockerfile` base to `mysql:5.7` with an explicit platform:

```dockerfile
FROM --platform=linux/amd64 mysql:5.7
```

`mysql:5.7` pulls and unpacks cleanly. The `--platform` in the `FROM` is what makes the
classic-vs-buildx build resolve the amd64 image (Compose's `platform:` field alone is not
honoured by every builder).

### 2. Composer blocks the intentionally-vulnerable `twig 1.19.0` — 1 benchmark

**044**

Recent Composer refuses to install packages named in security advisories, and this
challenge deliberately pins `twig/twig 1.19.0` (the vulnerable dependency **is** the
challenge target).

**Fix:** pin an older Composer in `app/Dockerfile` that predates advisory-blocking:

```dockerfile
COPY --from=composer:2.2 /usr/bin/composer /usr/bin/composer
```

### 3. PhantomJS removed from Debian — 3 benchmarks

**011, 012, 013** (XSS challenges: the app runs `phantomjs check.js` as the victim "bot")

PhantomJS was removed from all current Debian repos, so `apt-get install phantomjs` fails
— "no installation candidate" on `trixie` (011), and 404s on the EOL `buster` base
(012, 013).

**Fix:** vendor the official **PhantomJS 2.1.1** static binary and install it plus its
runtime libraries instead of `apt-get install phantomjs`.

- The binary is vendored per-benchmark as `src/phantomjs-2.1.1-linux-x86_64.tar.bz2`
  (sha256 `86dd9a4bf4aee45f1a84c9f61cf1947c1d6dce9b9e8d2a907105da7852460d2f`), extracted
  in the Dockerfile; runtime libs `libfontconfig1 libfreetype6` are apt-installed.
- **012 / 013** additionally repoint apt at `archive.debian.org` (their `buster` base is
  EOL and its mirrors 404).
- The base is pinned `--platform=linux/amd64` (the binary is x86_64-only).

> **Runtime caveat.** On Apple Silicon the x86_64 PhantomJS binary aborts under QEMU
> emulation at solve time (it runs on a native x86_64 host). The images build regardless
> — PhantomJS is only invoked when solving the challenge, never at build time.

## Metadata normalization (PR #2)

`level` was stored as a string (`"1"/"2"/"3"`) in 63 of the 104 manifests and as an int in
the rest. All 104 are now `int`, matching the value already used consistently in every
`benchmark.yaml` and the type the harness loader coerces to (`int(manifest["level"])`).
Only the `level` value changed — no other keys, formatting, ordering, or the derived flag
(which comes from the challenge name, not the JSON).

## Runtime caveats on Apple Silicon (local only)

amd64 images run under Colima's QEMU emulation, which is intermittently unstable:

- `mysql:8.0` can segfault (`exited (139)`) on start — usually succeeds on retry.
- The vendored PhantomJS binary aborts (see above).

Both are emulation artifacts and do not occur on a native x86_64 host (e.g. the evaluation
infrastructure).
