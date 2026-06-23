# dcmclient

**One static binary, 65 DICOM tools — first-class for people and for agents.** A
complete command-line DICOM toolbox *and* a self-describing MCP/agent surface,
from one tool registry. No SDK, no per-tool glue, no runtime dependencies. It
opens **any DICOM, from any vendor or country**: a built-in decoder for **every
transfer syntax** — **JPEG, JPEG-2000, JPEG-LS, RLE, JPEG-XL and HTJ2K** (the last
two most toolchains can't read), no codec plugins — and **every character set**
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

65 tools covering the whole job — inspect, convert, author, extract, edit,
de-identify, validate, and move studies over the network:

```console
# inspect — text / JSON / XML / semantic content / SR
$ dcmclient dcmdump     scan.dcm               # one element per line
$ dcmclient dcm2json    scan.dcm > scan.json   # DICOM JSON (PS3.18)
$ dcmclient dcm2xml     scan.dcm > scan.xml    # Native DICOM XML (PS3.19)
$ dcmclient dcm2content seg.dcm                # semantic JSON of SEG / RT / PS / waveform
$ dcmclient dsrdump     report.dcm             # Structured Report content tree

# extract pixels / signals — decode ANY transfer syntax
$ dcmclient dcm2raw     ct.dcm out.raw --json  # raw pixels + numpy sidecar
$ dcmclient dcm2img     ct.dcm out.png         # 8-bit PNG for display
$ dcmclient dcm2mpg     video.dcm out.mp4      # embedded video stream
$ dcmclient dcm2waveform ecg.dcm out.csv       # ECG / EEG samples

# convert — volumes (FSL / 3D Slicer / ITK / nnU-Net), codecs, WSI, FHIR
$ dcmclient dcmconv     --to jxl ct.dcm out.dcm # transcode (rle / jls / htj2k / jxl / …)
$ dcmclient dcm2nii     ct_series/ ct.nii.gz   # NIfTI — DWI .bval/.bvec, BIDS, --split batch
$ dcmclient dcm2nrrd    ct_series/ ct.nrrd     # NRRD (3D Slicer)
$ dcmclient dcm2mha     ct_series/ ct.mha      # MetaImage (ITK / nnU-Net)
$ dcmclient seg2nrrd    seg.dcm labels.seg.nrrd # SEG → 3D Slicer segmentation
$ dcmclient dcmresample ct_series/ iso.nii.gz --spacing 1   # isotropic resample
$ dcmclient dcm2tiff    slide.dcm region.tiff --level 0     # WSI pyramid → TIFF
$ dcmclient dcm2fhir    study/ > study.fhir.json # → FHIR R4 ImagingStudy

# author — SEG / parametric map / SR / KOS, or build from JSON / image / PDF
$ dcmclient mkseg       ct.dcm --mask labelmap.dcm --meta seg.json -o seg.dcm
$ dcmclient mkparamap   series/ --values adc.f32 --meta adc.json -o adc.dcm
$ dcmclient mkreport    findings.json report.dcm --series-from study.dcm   # TID 1500 SR
$ dcmclient img2dcm     photo.jpg out.dcm      # JPEG → Secondary Capture
$ dcmclient dcmencap    model.stl out.dcm      # wrap PDF / STL / CDA  ↔  dcmdecap unwraps

# measure / edit / de-identify / validate / sign
$ dcmclient dcmradiomics ct.dcm --mask roi.dcm --json   # IBSI radiomic features
$ dcmclient dcmodify     scan.dcm -i '(0010,0010)=Anon^Patient'   # byte-verbatim edit
$ dcmclient dcmdeident   study/*.dcm --out-dir deid/    # PS3.15 Basic Profile + all option profiles, UID remap
$ dcmclient dcmvalidate  scan.dcm                       # IOD conformance — Type 1/2 modules (nested), VM, enumerated values
$ dcmclient dsign        in.dcm signed.dcm --key key.pem --cert cert.pem   # PS3.15 signature

# network — DICOMweb: convenience verbs (search / pull / push / forward) or the raw
#           RS verbs (qido / wado / stow / ups); DIMSE: storescu / findscu / movescu /
#           storescp / echoscu / getscu (dashed aliases — store-scu, etc. — also work)
$ dcmclient search    https://pacs.example.com --patient-id 42 --json   # QIDO    (raw: dcmclient qido …)
$ dcmclient pull      https://pacs.example.com --study 1.2.3 -o ./out    # WADO-RS (raw: dcmclient wado …)
$ dcmclient storescu  PACS 11112 ./*.dcm --aec PACS
$ dcmclient findscu   PACS 11112 -S -k QueryRetrieveLevel=SERIES --json
$ dcmclient movescu   PACS 11112 --move-dest STORESCP -S -k StudyInstanceUID=1.2.3
$ dcmclient storescp  11112 --aet STORESCP --out-dir incoming/   # receive C-MOVE / C-STORE (the dest above)
```

The binary also answers to the shorter **`dcmcli`** (a built-in alias) — `dcmcli
dcm2json scan.dcm` is identical to the full name. And per-tool: `ln -s dcmclient
dcmdump` makes `dcmdump scan.dcm` work directly. Full list + every flag in the
**[tool reference](https://dcmkit.github.io/dcmclient/reference/)**.

## A self-describing agent surface

Every tool is declared once in a single registry. That one declaration *is* the
command-line parser, the `manifest` self-description, **and** the `mcp`
`tools/list` reply — so the agent surface can never drift from what the binary
actually does.

- **`mcp`** — a live [Model Context Protocol](https://modelcontextprotocol.io)
  server over stdio; any MCP client (Claude Desktop, Claude Code, your own
  runtime) discovers and calls all 65 tools with zero per-tool wiring.
- **`manifest`** — each tool's `inputSchema` + `outputSchema` as JSON, ready to
  drop into a function-calling tool definition. Filter by `readOnly` /
  `network` / `writesFiles` to expose only a safe subset.
- **JSON envelope** — `{tool, ok, exit_code, result, error}` on `--json`;
  `stdout` is machine output, `stderr` is human diagnostics; graduated exit
  codes separate bad input from runtime failure.
- **Agent Skill** — `docs/skill.md` is a task → tool map an agent loads to learn
  what the toolbox can do and which command to reach for, before it ever calls one.

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
$ ln -s dcmclient /usr/local/bin/dcmcli       # optional: the short `dcmcli` alias (Homebrew adds it for you)
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
