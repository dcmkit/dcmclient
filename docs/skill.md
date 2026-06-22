---
name: dcmclient
description: >-
  Drive the `dcmclient` CLI to run complete DICOM / medical-imaging workflows —
  query and pull studies from a PACS over DICOMweb or DIMSE, de-identify,
  convert to NIfTI / FHIR / TIFF, render images, author SEG / RT / SR / KOS
  objects, compute radiomics and RT-DVH, and digitally sign. Use this whenever
  a task touches DICOM files, a PACS / DICOMweb server, or any radiology,
  pathology (WSI), RT, or waveform (ECG/EEG) data — even when the request only
  implies it (e.g. "anonymize these scans", "make a NIfTI for the model",
  "pull the latest CT", "build a segmentation object"). One static binary; each
  step is one subcommand. Prefer it over hand-rolling DICOM parsing glue.
compatibility: Requires the `dcmclient` binary on PATH (or `dcmclient mcp` wired as an MCP server). No Python needed.
---

# dcmclient — DICOM workflows from the command line

`dcmclient` is one static binary exposing 60+ DICOM tools. Every tool follows
the same contract, so an agent can chain them without bespoke glue. This skill
maps **tasks → tools** and gives **end-to-end recipes**. It does not duplicate
the tool reference — get exact argument schemas at runtime from
`dcmclient manifest` (JSON, all tools) or `dcmclient <tool> --help`.

## The calling contract (read once)

- **`stdout` = machine output, `stderr` = human diagnostics.** Pass `--json` to
  status/network tools to get a `{tool, ok, exit_code, result, error}` envelope.
  Document tools (`dcm2json`, `dcm2xml`, `dcmdump`, `dcmvalidate`, `dcm2content`,
  `dsrdump`) write their structured artifact straight to stdout.
- **Branch on exit code, not stderr text:** `0` ok · `1` runtime/input error
  (also covers tool-level rejections like an unknown enum string) · `104` bad
  numeric value · `105` rejected/mutually-exclusive option · `106` required
  option missing. Other parser codes exist (e.g. `109` unknown option); treat
  any non-zero as failure and read `error` from the `--json` envelope.
- Network tools (`search`/`wado`/`stow`/`forward`, the `*-scu`) also surface the
  DICOM response status inside `result`.
- Many subcommands have familiar short aliases: `qido`≡`search`, `wado`≡`pull`,
  `stow`≡`push`. Either name works.

## Capability map (task → tool)

| You need to… | Tool(s) |
|---|---|
| Search a PACS / DICOMweb | `search` (QIDO) · `find-scu` (C-FIND) |
| Pull studies | `wado` (WADO-RS) · `wado-uri` · `get-scu`/`move-scu` (C-GET/MOVE) |
| Push / route | `stow` (STOW-RS) · `store-scu` (C-STORE) · `forward` (web↔DIMSE) |
| Delete (DICOMweb) | `delete` |
| Run a receiver | `listen` · `store-scp` |
| Verify connectivity | `echo-scu` |
| Workflow / status SCUs (DIMSE-N) | `mpps-scu` · `ups-scu`/`ups` · `stgcmt-scu` · `ian-scu` · `print-scu` · `term-scu` |
| **De-identify** | `dcmdeident` (PS3.15 profiles) |
| Inspect metadata | `dcmdump` · `dcm2json` · `dcm2xml` · `dcmprobe` |
| Inspect *semantics* (SEG/RT/PS/SR auto-detect) | `dcm2content` |
| Render to image / video | `dcm2img` · `dcm2mpg` |
| Decode pixels for AI | `dcm2raw` (raw native ints) · `dcm2nii` (3-D NIfTI) |
| Edit tags | `dcmodify` |
| Transcode / change transfer syntax | `dcmconv` |
| Compare two objects | `dcmicmp` |
| Validate IOD conformance | `dcmvalidate` |
| Read SR documents | `dsrdump` · `dsr2html` · `dsr2xml` |
| Build / round-trip SR from text | `mksr` · `xml2dsr` |
| Build DICOM from JSON / XML / image / dump | `json2dcm` · `xml2dcm` · `img2dcm` · `dump2dcm` |
| Build a DICOMDIR (media interchange) | `dcmmkdir` |
| → NIfTI / BIDS | `dcm2nii` · `seg2nii` (SEG→labelmap NIfTI) |
| → FHIR ImagingStudy | `dcm2fhir` |
| → TIFF / WSI ↔ DICOM | `dcm2tiff` · `tiff2dcm` |
| Resample a volume | `dcmresample` |
| Radiomics features | `dcmradiomics` |
| RT dose-volume histogram | `dcmdvh` |
| Author SEG / paramap / SR / report / KOS | `mkseg` · `mkparamap` · `mksr` · `mkreport` · `mkkos` |
| Waveforms (ECG/EEG) | `dcm2waveform` |
| Digitally sign / verify | `dsign` |
| Encapsulate / extract PDF, STL, … | `dcmencap` · `dcmdecap` · `pdf2dcm` · `dcm2pdf` |

## The core loop (canonical recipe)

Each step is one tool call. Under MCP, issue the same calls as `tools/call`
and read each JSON result to decide the next step.

```console
# find the study (QIDO over DICOMweb)
dcmclient search --server https://pacs.example.com --patient-id 42 --modality CT --json
# pull it locally (WADO-RS)
dcmclient wado --server https://pacs.example.com --study 1.2.840.. -o ./study
# de-identify BEFORE anything downstream (see guardrails)
dcmclient dcmdeident ./study/*.dcm --out-dir ./deid
# build a 3-D NIfTI volume for a model
dcmclient dcm2nii ./deid ct.nii.gz
```

DIMSE variant: swap the first two steps for `find-scu` then `get-scu`/`move-scu`
against a classic PACS — same downstream steps.

## Deeper recipes by domain

Read the matching reference for multi-step recipes, real flags, and gotchas —
load only the one the task needs.

| When the task is… | Read |
|---|---|
| query / retrieve / store / route against a PACS or DICOMweb, or DIMSE-N workflow messaging | [references/networking.md](references/networking.md) |
| feeding imaging into a model — volumes, NIfTI, raw pixels, resampling, radiomics | [references/ai-prep.md](references/ai-prep.md) |
| reading or authoring a semantic object — SEG, parametric map, RT plan/dose/DVH, SR, KOS | [references/structured-objects.md](references/structured-objects.md) |
| viewing pixels, transcoding (e.g. → JPH/HTJ2K), converting out (FHIR/TIFF/WSI/waveform), encapsulating, or signing | [references/render-export.md](references/render-export.md) |

## Guardrails (do not skip)

- **De-identify before egress.** Run `dcmdeident` before sending data to any
  external service, FHIR endpoint, or third-party model API. Patient data leaves
  the building the moment you `stow`/`forward`/`store-scu` to a remote node.
- **Validate authored objects** (`dcmvalidate`) before sending SEG/RT/SR/paramap
  back to a PACS — a malformed IOD can be silently rejected or corrupt a study.
- **Prefer `--json` for any step whose output feeds the next decision** — parse
  the envelope, branch on `ok`/`exit_code`, don't scrape human text.
- **Network tools always need an explicit endpoint** (`--server` for DICOMweb,
  AE/host/port for DIMSE). Never assume a default PACS.
- **WSI writing is TILED_FULL only**; vendor SVS/NDPI are read-side concerns,
  not `tiff2dcm` inputs.

## Going deeper

- Exact argument schemas, examples, and read-only/network/writes-files flags for
  every tool: `dcmclient manifest` (full JSON registry) or `dcmclient <tool> --help`.
- Wiring this as a live MCP server (Claude Desktop/Code, or any framework via the
  raw manifest): see the product's **Agent integration** doc.
- pydcm (the Python package) covers the same imaging engine for agents that write
  Python instead of shelling out — use that surface when the task is "write
  Python", this one when the task is "run a command".
