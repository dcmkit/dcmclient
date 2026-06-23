# Install

`dcmclient` is a single static binary with no runtime dependencies. Pick
whichever channel fits you — all four platforms (macOS arm64 / x86_64, Linux
x86_64 / aarch64, glibc 2.28+ → RHEL 8+ / Ubuntu 20.04+) are covered.

## Homebrew (macOS &amp; Linux)

```console
$ brew install dcmkit/tap/dcmclient
$ dcmclient --help
```

`brew upgrade dcmclient` keeps it current. This is the recommended path — it
picks the right build for your machine and puts it on your `PATH`.

## Direct download

Grab the binary for your platform from the
[releases page](https://github.com/dcmkit/dcmclient/releases) (or the URL
below), drop it on your `PATH`:

```console
$ curl -L -o dcmclient https://github.com/dcmkit/dcmclient/releases/latest/download/dcmclient-macos-arm64
$ install -m 755 dcmclient /usr/local/bin/
$ dcmclient --help
```

Asset names: `dcmclient-macos-arm64`, `dcmclient-macos-x86_64`,
`dcmclient-linux-aarch64`, `dcmclient-linux-x86_64`. The macOS builds are
Developer-ID signed and notarized, so they run without a Gatekeeper prompt.

## Container (for agents &amp; CI)

For sandboxed agent runtimes and CI that can't drop a binary on `PATH`, run the
container instead — the binary is the entrypoint, so any tool name works as the
command:

```console
$ docker run --rm -v "$PWD:/work" ghcr.io/dcmkit/dcmclient dcm2raw /work/in.dcm /work/out.raw --json
$ docker run --rm -i ghcr.io/dcmkit/dcmclient mcp        # live MCP server over stdio
$ docker run --rm ghcr.io/dcmkit/dcmclient manifest      # agent tool self-discovery
```

Mount the directory holding the DICOM you want reachable (`-v host:container`),
and pass `-i` when running `mcp` so the agent can speak JSON-RPC over stdin.

## Per-tool commands (optional)

The single binary dispatches on its own name. Symlink it to any tool name and
that name becomes a standalone command — no extra download:

```console
$ ln -s dcmclient /usr/local/bin/dcmdump
$ dcmdump scan.dcm            # identical to: dcmclient dcmdump scan.dcm
```

Familiar spellings from other DICOM toolchains work too — `ln -s dcmclient
echoscu` dispatches to `echo-scu`.

## Verify

```console
$ dcmclient --help            # full command list
$ dcmclient manifest | head   # the agent-facing self-description
```

!!! note
    The binary embeds its DICOM engine and codecs; there is nothing else to
    install. Building from source is not a public workflow — parts of the
    native engine are not open source.
