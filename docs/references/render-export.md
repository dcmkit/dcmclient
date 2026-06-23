# Render & export — images, video, transcode, interop, signing

Read this to **look at pixels**, change a transfer syntax, convert out of DICOM,
or sign an object.

## Render to a viewable image / video — `dcm2img`

```console
dcmclient dcm2img --format png  --frame 48 scan.dcm out.png    # one frame, lossless raster
dcmclient dcm2img --format mp4             cine.dcm out.mp4    # multi-frame → H.264 video
dcmclient dcm2img --data-uri    --frame 1  scan.dcm           # base64 image to stdout (MCP vision block)
```
- **Lossless + viewable = `png`** (8-bit windowed display) or `png16`/`tiff`
  (full native depth). **Baseline `jpeg` is lossy** — use it only for a quick
  `.jpg`; `jpeg16`/`jpeg12`/`jls` are lossless but most viewers can't open them.
- Multi-frame: `gif`/`webp` (lossless animation), `mp4`/`mov` (lossy 8-bit,
  full-res), `mkv` (FFV1, **lossless**, keeps native depth — needs VLC).
- Useful flags: `--frame N` (1-based), `--window c,w` (VOI override / one window
  for a whole cine), `--quality 1..100` (display jpeg), `--scale` (cap dimension),
  `--fps` (gif/webp), `--json` (status with width/height/bytes).
- `dcm2mpg` makes a bit-exact stream copy of an embedded-video object.

## Transcode / change transfer syntax — `dcmconv`

```console
dcmclient dcmconv --to htj2k       in.dcm out.dcm   # JPH / HTJ2K lossless (1.2.840.10008.1.2.4.201)
dcmclient dcmconv --to jls         in.dcm out.dcm   # JPEG-LS lossless
dcmclient dcmconv --to jpeg-lossless in.dcm out.dcm
dcmclient dcmconv --to evr-le      in.dcm out.dcm   # decompress to uncompressed VR-LE
```
`--to` (alias `--write-xfer`) takes a UID or an alias: `j2k` · `j2k-lossy` ·
`jls` · `jls-lossy` · `htj2k` · `htj2k-lossy` · `jxl` · `jxl-lossy` · `jpeg`
(baseline) · `jpeg-extended` · `jpeg-lossless` · `jpeg-lossless-sv1` · `rle` ·
`ivr-le` · `evr-le` · `deflated`. Verify the result with `dcmprobe --json`
(shows `transfer_syntax`).

**Video transfer syntaxes (MPEG-2/H.264/HEVC, `.100`–`.108`) are out of
`dcmconv`'s scope — it neither reads nor writes them:**
- To extract the stream from an *existing* embedded-video DICOM, use `dcm2mpg`
  (bit-exact copy). On a non-video file it exits non-zero with
  `not an embedded-video DICOM … use dcm2img`.
- To *view* any multi-frame object as video, use `dcm2img --format mp4/mkv/...`
  (renders frames → a standalone video file; this is NOT a video-TS DICOM).
- There is no encode-frames-*into*-a-video-DICOM path; authoring an embedded
  H.264/HEVC DICOM is not supported.

## Convert out of DICOM

```console
dcmclient dcm2fhir     study_dir/ study.fhir.json   # → FHIR ImagingStudy
dcmclient dcm2waveform ecg.dcm    ecg.csv           # ECG/EEG → CSV (--raw / --json)
dcmclient dcm2tiff     slide.dcm  ./tiff-out        # WSI DICOM → pyramidal TIFF
dcmclient tiff2dcm     slide.tif  ./wsi-out         # pyramidal TIFF → DICOM WSI
```
**WSI writing is TILED_FULL only**; vendor SVS/NDPI are read-side concerns, not
`tiff2dcm` inputs.

## Encapsulate / extract

`dcmencap` / `dcmdecap` (generic encapsulated documents — PDF, STL, …) ·
`pdf2dcm` / `dcm2pdf` (PDF specifically).

## Sign / verify — `dsign`

```console
dcmclient dsign in.dcm signed.dcm --key key.pem --cert cert.pem   # default MAC, or --mac SHA512
dcmclient dsign --verify signed.dcm --json
```
