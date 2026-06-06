# publish-adapter

A reusable GitHub Action that publishes a **pre-built** [Criteria](https://github.com/brokenbots/criteria)
adapter binary as a signed OCI artifact.

## Scope: publish only

Building the adapter binary is **the adapter's own job** — every adapter has its
own toolchain (Bun, Nuitka, `go build`, …) and only the adapter knows how to
build itself. So this action does **not** build anything adapter-specific. It
takes a binary your CI already built and performs the publish steps:

1. `--emit-manifest` → extract `adapter.yaml` from the binary,
2. construct the OCI artifact (`oras`),
3. attach a cosign signature (when `sign-key` is set),
4. push to the registry.

The same applies to container-image mode: what goes in the image is
adapter-specific (its `Dockerfile`/runtime deps), so image building stays with
the adapter too. This action is deliberately the build-agnostic publish half.

## Usage

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
    steps:
      - uses: actions/checkout@v4

      # 1. Build your adapter however your adapter builds (toolchain-specific).
      - name: Build adapter
        run: bun build --compile index.ts --outfile out/adapter   # example

      # 2. Authenticate to your registry.
      - name: Log in to GHCR
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin

      # 3. Publish the pre-built binary. Signs keyless by default (id-token: write).
      - uses: brokenbots/publish-adapter@v0
        with:
          binary: out/adapter
          registry: ghcr.io/${{ github.repository_owner }}/my-adapter:${{ github.ref_name }}
          # keyless: "true"   # default; set "false" to publish unsigned
          # sign-key: ...      # optional: explicit-key signing instead of keyless
          # image: ...         # optional: record an already-pushed runnable image
```

### Inputs

| Input | Required | Description |
|---|---|---|
| `binary` | yes | Path to the pre-built adapter binary. |
| `registry` | yes | Fully-qualified OCI reference, e.g. `ghcr.io/org/name:1.2.3`. |
| `keyless` | no | Sign keyless via Sigstore Fulcio using the workflow's ambient OIDC identity. **Default `true`** — requires `id-token: write`. Set `"false"` to publish unsigned. |
| `sign-key` | no | Path to a PEM Ed25519 cosign key. When set, signs with that explicit key instead of keyless. |
| `image` | no | Reference of an already-built, already-pushed runnable container image to record in the manifest (e.g. `ghcr.io/org/name:1.2.3-image`). The adapter's CI builds + pushes it; this action records the resolved digest. Default: artifact-only. |
| `criteria-ref` | no | Git ref of `brokenbots/criteria` used to build the CLI (default `adapter-v2`). See below. |

## Signing

By default (`keyless: true`) the action signs the artifact **keyless** via
Sigstore — the workflow's GitHub OIDC identity is certified by Fulcio and the
cosign signature is attached as an OCI referrer. This needs `id-token: write` on
the job (the starter templates already grant it). Power users who manage their
own keys can set `sign-key` instead; it takes precedence over keyless. Set
`keyless: "false"` with no `sign-key` to publish an unsigned artifact (not
recommended outside local experiments).

## Switching to a released CLI (do this after the v2 release)

> **TEMPORARY — tracked follow-up.** This action currently builds the `criteria`
> CLI **from source** (`git clone` + `go build`) on every run, pinned to the
> `criteria-ref` input. This is necessary because `criteria` is a multi-module
> repo: `go install …/cmd/criteria@<ref>` can't resolve the `sdk`/`workflow`
> `v0.0.0` replaces from the module proxy, so a from-source build (which uses the
> repo's `go.work`) is the only option until a release exists.
>
> **Once the criteria v2 release ships CLI binaries**, replace the
> *"Build criteria CLI from source"* step in [`action.yml`](action.yml) with a
> download of the pinned release binary, and replace the `criteria-ref` input
> with a `criteria-version` input. This removes the per-run build cost and the
> dependency on the `adapter-v2` branch. The step is commented in `action.yml`
> as `TEMPORARY (build-from-source)` to make it easy to find.

## Not yet supported (fast-follows)

- **Multi-arch** OCI index (currently single-platform — pass one binary).
- **Keyless** (Sigstore OIDC) signing — currently key-mode via `sign-key`.
- **`with_image`** container-image publishing.
