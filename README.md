# dcmclient

**One static binary, 61 DICOM tools — first-class for people and for agents.** A
complete command-line DICOM toolbox *and* a self-describing MCP/agent surface,
from one tool registry. No SDK, no per-tool glue, no runtime dependencies. It
opens **any DICOM, from any vendor or country** — **every transfer syntax**
(incl. JPEG-XL / HTJ2K most toolchains can't read) and **every character set**
(Japanese / Korean / Chinese multibyte, all ISO 2022) — where strict tools choke.

```console
# drive it yourself
$ dcmclient dcmdump scan.dcm                   # inspect — one element per line
$ dcmclient dcmconv --to jxl ct.dcm out.dcm    # transcode (incl. JPEG-XL / HTJ2K)
$ dcmclient dcmvalidate scan.dcm               # IOD conformance check
$ dcmclient search https://pacs.example.com --patient-id 42 --json   # query a PACS

# …or hand it to an agent
$ dcmclient mcp                # serve every tool to an MCP client over stdio
$ dcmclient manifest           # per-tool inputSchema + outputSchema as JSON
```

> **Not a medical device.** Not intended or cleared for clinical or diagnostic
> use — outputs are for research and engineering only.

→ **[Quickstart](https://dcmkit.github.io/dcmclient/quickstart/)** ·
**[Agent integration](https://dcmkit.github.io/dcmclient/agents/)** ·
**[Tool reference](https://dcmkit.github.io/dcmclient/reference/)** (every flag of every tool)

## A complete command-line toolbox

61 tools covering the whole job — inspect, convert, author, extract, edit,
de-identify, validate, and move studies over the network:

```console
# inspect — as text / JSON / XML / SR tree
$ dcmclient dcmdump   scan.dcm                 # one element per line
$ dcmclient dcm2json  scan.dcm > scan.json     # DICOM JSON (PS3.18)
$ dcmclient dcm2xml   scan.dcm > scan.xml      # Native DICOM XML (PS3.19)
$ dcmclient dsrdump   report.dcm              # Structured Report content tree

# convert & transcode — every codec built in (JPEG-2000 / JPEG-LS / RLE / JPEG-XL / HTJ2K)
$ dcmclient dcmconv   --to jxl ct.dcm out.dcm
$ dcmclient dcm2nii   ct_series/ ct.nii.gz     # → NIfTI
$ dcmclient pdf2dcm   report.pdf out.dcm       # wrap a PDF as Encapsulated PDF

# edit / de-identify / validate
$ dcmclient dcmodify    scan.dcm -i '(0010,0010)=Anon^Patient'
$ dcmclient dcmdeident  study/*.dcm --out-dir deid/    # PS3.15 Annex E
$ dcmclient dcmvalidate scan.dcm                       # IOD conformance

# networking — DICOMweb (search / pull / push) + DIMSE (echo / store / find / get / move)
$ dcmclient pull      https://pacs.example.com --study 1.2.3 -o ./out
$ dcmclient store-scu PACS 11112 ./*.dcm --aec PACS
$ dcmclient find-scu  PACS 11112 -S -k QueryRetrieveLevel=SERIES --json
```

Prefer per-tool commands? One symlink gives each its own name — `ln -s dcmclient
dcmdump` makes `dcmdump scan.dcm` work. Full list + every flag in the
**[tool reference](https://dcmkit.github.io/dcmclient/reference/)**.

## A self-describing agent surface

Every tool is declared once in a single registry. That one declaration *is* the
command-line parser, the `manifest` self-description, **and** the `mcp`
`tools/list` reply — so the agent surface can never drift from what the binary
actually does.

- **`mcp`** — a live [Model Context Protocol](https://modelcontextprotocol.io)
  server over stdio; any MCP client (Claude Desktop, Claude Code, your own
  runtime) discovers and calls all 61 tools with zero per-tool wiring.
- **`manifest`** — each tool's `inputSchema` + `outputSchema` as JSON, ready to
  drop into a function-calling tool definition. Filter by `readOnly` /
  `network` / `writesFiles` to expose only a safe subset.
- **JSON envelope** — `{tool, ok, exit_code, result, error}` on `--json`;
  `stdout` is machine output, `stderr` is human diagnostics; graduated exit
  codes separate bad input from runtime failure.

## Install

**Homebrew** (macOS &amp; Linux) — the recommended path:

```console
$ brew install dcmkit/tap/dcmclient
$ dcmclient --help
```

**Direct download** — grab the binary for your platform from the
[releases page](https://github.com/dcmkit/dcmclient/releases) and put it on your
`PATH` (macOS builds are Developer-ID signed &amp; notarized):

```console
# macOS (arm64); swap the asset for your platform:
#   dcmclient-macos-arm64 · dcmclient-macos-x86_64
#   dcmclient-linux-aarch64 · dcmclient-linux-x86_64  (glibc 2.28+)
$ curl -L -o dcmclient https://github.com/dcmkit/dcmclient/releases/latest/download/dcmclient-macos-arm64
$ install -m 755 dcmclient /usr/local/bin/
$ dcmclient --help
```

**Container** (sandboxed agents / CI) — the binary is the entrypoint:

```console
$ docker run --rm -i ghcr.io/dcmkit/dcmclient mcp        # MCP server over stdio
$ docker run --rm -v "$PWD:/work" ghcr.io/dcmkit/dcmclient dcm2json /work/scan.dcm
```

One symlink gives each tool its own name — `ln -s dcmclient dcmdump` makes
`dcmdump scan.dcm` work, with no extra binary to ship.

## Documentation

Full docs — quickstart, agent integration, and the generated tool reference —
at **[dcmkit.github.io/dcmclient](https://dcmkit.github.io/dcmclient/)**.

## License

**Apache-2.0** (see [LICENSE](LICENSE) / [NOTICE](NOTICE)). The DICOM engine
ships as a compiled binary; third-party components linked into it are listed in
[THIRD-PARTY-LICENSES](THIRD-PARTY-LICENSES) — all permissive except **FFmpeg**,
which is included under **LGPL-2.1** with a §6 relink offer.
