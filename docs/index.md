# dcmclient

**One static binary, 61 DICOM tools — first-class for people and for agents.**
dcmclient is a complete command-line DICOM toolbox: query and move studies over
**DICOMweb and DIMSE**, and inspect, convert, author, extract, edit, de-identify
and validate **Part-10 files**. It opens **any DICOM, from any vendor or
country** — **every transfer syntax** (JPEG, JPEG-2000, JPEG-LS, RLE, video, and
the modern **JPEG-XL / HTJ2K** most toolchains still can't read) and **every
character set** (Japanese / Korean / Chinese multibyte, all ISO 2022) — where
strict tools choke. The very same tools are a self-describing agent surface — a
built-in **MCP server** and a machine-readable **`manifest`** with per-tool JSON
schemas — so an LLM drives the whole roster with zero glue. One registry, two
ways to run it.

!!! tip "No “unsupported” file"
    **Every pixel format, every encoding, built in.** Compressed or not —
    including JPEG-XL and HTJ2K — decodes with no plugins; text from any vendor
    or locale reads back as correct UTF-8, handled adaptively where a strict
    reader would error or garble it.

```console
# drive it yourself
$ dcmclient search https://pacs.example.com --patient-id 42 --json
$ dcmclient dcm2nii ct_series/ ct.nii.gz

# …or hand it to an agent
$ dcmclient mcp                # serve every tool to an MCP client over stdio
$ dcmclient manifest           # per-tool inputSchema + outputSchema as JSON
```

!!! warning "Not a medical device"
    Not intended or cleared for clinical or diagnostic use — outputs are for
    research and engineering only.

## A complete command-line toolbox

One binary covers the whole job — no SDK, no per-tool glue, no runtime
dependencies:

- **Networking** — QIDO / WADO / STOW / DELETE over DICOMweb, the full
  C-ECHO / C-STORE / C-FIND / C-GET / C-MOVE DIMSE set plus a Storage SCP, and
  pull→push migration between archives.
- **Local file tools** — inspect (dump / JSON / XML / semantic content),
  render &amp; extract (raw pixels / PNG / video / waveforms / NIfTI), convert &amp;
  transcode (every codec, including JPEG-XL and HTJ2K), author (SEG / parametric
  map / SR / KOS), and edit / de-identify / validate.

```console
$ dcmclient store-scu PACS 11112 ./*.dcm --aec PACS
$ dcmclient dcmdeident study/*.dcm --out-dir deid/
$ dcmclient dcmvalidate ct.dcm --json
```

Prefer per-tool commands? One symlink gives each tool its own name —
`ln -s dcmclient dcmdump` makes `dcmdump scan.dcm` work, with no extra binary to
ship.

## A self-describing agent surface

Every tool is declared once in a single registry. That one declaration *is* the
command-line parser, the `manifest` self-description, **and** the `mcp`
`tools/list` reply — so the agent surface can never drift from what the binary
actually does:

- **`mcp`** — a live [Model Context Protocol](agents.md) server over stdio; any
  MCP client (Claude Desktop, Claude Code, your own runtime) discovers and calls
  all 61 tools with zero per-tool wiring.
- **`manifest`** — each tool's `inputSchema` + `outputSchema` as JSON, ready to
  drop into a function-calling tool definition. Filter by `readOnly` / `network`
  / `writesFiles` to expose only a safe subset.
- **JSON envelope** — `{tool, ok, exit_code, result, error}` on `--json`;
  `stdout` is machine output, `stderr` is human diagnostics; graduated exit codes
  separate bad input from runtime failure.

→ See [Agent integration](agents.md) for MCP client config and an end-to-end run.

## Where to go

- [Install](install.md) — download the binary, or run the container (agents / CI)
- [Quickstart](quickstart.md) — networking, local tools, the agent surface
- [Agent integration](agents.md) — MCP config, `manifest`, an end-to-end run
- [Tool reference](reference.md) — every flag of every tool, generated from
  `manifest` so it cannot drift from the binary
