# CLI quickstart

`dcmclient` is one static binary (also installed as the shorter **`dcmcli`** —
`dcmcli <subcommand>` is identical). Every subcommand is self-describing, and the
whole roster is agent-discoverable over `manifest` and `mcp`.

```console
$ dcmclient --help            # the full command list  (or: dcmcli --help)
$ dcmclient dcmdump --help    # per-tool flags
```

## Networking

```console
# DICOMweb — convenience verbs, or the raw RS verbs (qido / wado / stow)
$ dcmclient search https://pacs.example.com --patient-id 42 --json
$ dcmclient pull   https://pacs.example.com --study 1.2.3 -o ./out
$ dcmclient push   https://pacs.example.com ./series/*.dcm --study 1.2.3

# DIMSE — verification / storage / query-retrieve (conventional tool names; dashed aliases like echo-scu also work)
$ dcmclient echoscu   PACS 4242 --aec PACS
$ dcmclient storescu  PACS 11112 ./*.dcm --aec PACS
$ dcmclient findscu   PACS 11112 -S -k QueryRetrieveLevel=SERIES --json
$ dcmclient getscu    PACS 11112 -S -k StudyInstanceUID=1.2.3 --out-dir out/   # C-GET pull
$ dcmclient movescu   PACS 11112 --move-dest STORESCP -S -k StudyInstanceUID=1.2.3  # C-MOVE to a third party
$ dcmclient storescp  11112 --aet STORESCP --out-dir incoming/                 # receive the C-MOVE (or any C-STORE)
$ dcmclient termscu   PACS 4242 --aec PACS                                     # connectivity probe (open + release)

# DIMSE-N workflow services — MPPS, print, storage commitment, UPS, IAN
$ dcmclient mpps-scu  PACS 11112 --action create --dataset-file step.json
$ dcmclient stgcmt-scu PACS 11112 --dataset-file commit.json --transaction-uid 1.2.3.4
```

DICOMweb has both the convenience verbs above and the raw RS verbs as their own
tools (`qido` / `wado` / `wado-uri` / `stow` / `ups`, plus `delete` to remove a
study / series / instance); DIMSE covers the full classic set (`echo` / `store`
/ `find` / `get` / `move` SCU + `store-scp`, plus the `term-scu` connectivity
probe) and the **DIMSE-N** services (`mpps-scu`, `printscu`, `stgcmt-scu`,
`upsscu`, `ian-scu`). Every networked subcommand accepts `--profile <name>` to fill
`--server` / `--base-path` / `--auth` from a saved registry
(`dcmclient config add …`), plus AWS SigV4 (`--aws-*` / `$AWS_*`) and OAuth2
(`--oauth2-*` / `$DCMCLIENT_OAUTH2_*`).

## Local file tools

The 64-tool roster groups into eleven kinds of operation — at least one example of
each below. The generated [tool reference](reference.md) has every flag of every
tool.

**Inspect** — read a file as text / JSON / XML / semantic content

```console
$ dcmclient dcmdump    scan.dcm                 # one element per line
$ dcmclient dcm2json   scan.dcm > scan.json     # DICOM JSON (PS3.18 §F)
$ dcmclient dcm2xml    scan.dcm > scan.xml      # Native DICOM XML (PS3.19)
$ dcmclient dcm2content seg.dcm                 # semantic JSON of SEG/RT/PS/Waveform
$ dcmclient dsrdump    report.dcm               # SR content tree as text
```

!!! tip "Vendor character sets decode where strict readers fail"
    Text values come out as correct UTF-8 regardless of how the source declared
    its `SpecificCharacterSet` — the native text engine handles the full DICOM
    range (Latin / Cyrillic / Greek / Arabic / Hebrew / Thai, **Japanese**
    Shift-JIS + ISO 2022 IR 87 / 159, **Korean** EUC-KR, **Chinese**
    GB18030 / GBK, UTF-8) and decodes it **adaptively** — lenient charset-alias
    matching, tolerant of malformed ISO 2022 escapes, with a fault-tolerant
    fallback. The messy real-world vendor exports that a strict iconv-based reader
    rejects, `dcmdump` / `dcm2json` / `dcm2xml` read correctly.

**Render &amp; extract pixels / signals**

```console
$ dcmclient dcm2raw   ct.dcm out.raw --json     # decode ANY transfer syntax → raw pixels + numpy sidecar
$ dcmclient dcm2img   ct.dcm out.png            # 8-bit PNG/PNM for display
$ dcmclient dcm2mpg   video.dcm out.mp4         # extract an embedded video stream
$ dcmclient dcm2waveform ecg.dcm out.csv        # ECG/EEG samples → CSV / WAV
```

**Convert &amp; transcode**

```console
$ dcmclient dcmconv   --to j2k ct.dcm out.dcm   # transcode (also: rle / jls / htj2k / jxl / …)
$ dcmclient dcm2nii   ct_series/ ct.nii.gz      # validated 3-D NIfTI
$ dcmclient dcm2nii   --4d  dce_series/ dyn.nii.gz   # 4-D dynamic stack (time / echo / phase)
$ dcmclient dcm2nii   --dwi dti_series/ dwi.nii.gz   # 4-D DWI stack + FSL .bval/.bvec
$ dcmclient dcm2nii   --split study_dir/ out/   # batch: one .nii.gz per series, -f names them (dcm2niix-style)
$ dcmclient dcm2nrrd  ct_series/ ct.nrrd        # NRRD (3D Slicer; --gzip; double-faithful space)
$ dcmclient dcm2mha   ct_series/ ct.mha         # single-file MetaImage (ITK / nnU-Net; --compress)
$ dcmclient img2dcm   photo.jpg out.dcm         # JPEG → Secondary Capture
$ dcmclient tiff2dcm  slide.tif out_dir/        # pyramidal TIFF → DICOM WSI
$ dcmclient pdf2dcm   report.pdf out.dcm        # wrap a PDF  ↔  dcm2pdf extracts it back
$ dcmclient dcm2pdf   report.dcm report.pdf     #   …the reverse: pull the embedded document out
$ dcmclient dcmencap  model.stl out.dcm         # wrap PDF/CDA/STL/OBJ/MTL  ↔  dcmdecap unwraps
$ dcmclient dcmdecap  report.dcm out.pdf        #   …the reverse: extract any encapsulated payload
$ dcmclient json2dcm  scan.json out.dcm         # DICOM JSON Model → Part-10 (inverse of dcm2json)
$ dcmclient dump2dcm  edited.txt out.dcm        # an (edited) dcmdump text listing → Part-10
$ dcmclient dsr2xml   report.dcm report.xml     # SR content tree → XML  ↔  xml2dsr rebuilds the SR
$ dcmclient seg2nii   seg.dcm labels.nii.gz     # rasterize a SEG → labelmap NIfTI
$ dcmclient seg2nrrd  seg.dcm labels.seg.nrrd   # rasterize a SEG → 3D Slicer .seg.nrrd (named segments)
$ dcmclient dcmresample ct_series/ iso.nii.gz --spacing 1   # resample onto a 1 mm isotropic grid
$ dcmclient dcm2tiff  slide.dcm region.tiff --level 0 --region 10000,8000,1024,1024  # WSI pyramid region → TIFF
$ dcmclient dcm2fhir  study/ > study.fhir.json  # whole study → FHIR R4 ImagingStudy
```

!!! tip "Every codec built in — including JPEG-XL and HTJ2K"
    `dcmconv` encodes **and** decodes every DICOM transfer syntax out of the box,
    with no plugins or external codec packages — **JPEG-2000** (`.4.90/.91`),
    **JPEG-LS** (`.4.80/.81`), **RLE**, baseline / lossless **JPEG**, and the
    newest **JPEG-XL** (`.4.110/.111`) and **HTJ2K / High-Throughput JPEG 2000**
    (`.4.201–.203`). JPEG-XL and HTJ2K in particular are syntaxes most DICOM
    toolchains still don't ship, so dcmclient is often the only tool that can
    read or produce them:

    ```console
    # encode to the modern codecs
    $ dcmclient dcmconv --to jxl    ct.dcm ct_jxl.dcm        # JPEG-XL (lossless)
    $ dcmclient dcmconv --to htj2k  ct.dcm ct_htj2k.dcm      # HTJ2K / jph

    # decode them anywhere — dcm2raw / dcm2img read ANY transfer syntax
    $ dcmclient dcm2raw ct_jxl.dcm   pixels.raw --json
    $ dcmclient dcm2img ct_htj2k.dcm preview.png
    ```

**Author derived objects**

```console
$ dcmclient mkseg     ct.dcm --mask labelmap.dcm --meta seg.json -o out-seg.dcm   # coded SEG
$ dcmclient mkparamap series/ --values adc.f32 --meta adc.json -o adc_map.dcm     # Parametric Map
$ dcmclient mkreport  findings.json report.dcm --series-from study.dcm            # TID 1500 SR
$ dcmclient mksr      report.json -o report-sr.dcm                                # arbitrary SR content tree
$ dcmclient mkkos     keyimages.json kos.dcm --series-from study.dcm              # Key Object Selection
```

**Edit, de-identify &amp; validate**

```console
$ dcmclient dcmodify    scan.dcm -i '(0010,0010)=Anon^Patient'   # byte-verbatim edit
$ dcmclient dcmdeident  study/*.dcm --out-dir deid/            # PS3.15 de-identification
$ dcmclient dsign       in.dcm signed.dcm --key key.pem --cert cert.pem   # PS3.15 digital signature
$ dcmclient dcmradiomics ct.dcm --mask roi.dcm --json         # IBSI radiomic features
$ dcmclient dcmdvh      rtstruct.dcm rtdose.dcm --roi 1       # RT dose-volume histogram
$ dcmclient dcmprobe    scan.dcm                              # is it valid Part-10?
$ dcmclient dcmvalidate scan.dcm                             # IOD conformance
$ dcmclient dcmicmp     original.dcm compressed.dcm           # pixel-difference metrics
```

A directory of instances → a DICOMDIR-indexed file-set (PS3.10 media):

```console
$ dcmclient dcmmkdir study/ --output /media/dicom
```

!!! tip "De-identification — the full PS3.15 Annex E profile, configurable"
    `dcmdeident` implements the **PS3.15 Annex E Basic Profile** with every
    retain / clean option column as a flag — keep exactly what your use case
    allows and remove the rest. A batch shares one session, so UIDs are
    **remapped consistently** across the whole study, and it **never edits in
    place**:

    ```console
    # de-identify a study but keep dates and patient characteristics for research,
    # shift nothing, drop private tags, consistent UID remap across the batch:
    $ dcmclient dcmdeident study/*.dcm --out-dir deid/ \
          --retain-dates --retain-patient-chars

    # or shift every date by a fixed offset (consistent across the batch):
    $ dcmclient dcmdeident study/*.dcm --out-dir deid/ --shift-dates -3650 --json
    ```

    Options include `--retain-uids` / `--retain-device-id` /
    `--retain-institution-id` / `--retain-private` / `--clean-descriptors` /
    `--clean-graphics` — see the [tool reference](reference.md).

!!! tip "Validation — IOD conformance, agent-readable"
    `dcmvalidate` goes past VR / value-length / dictionary checks to full
    **IOD conformance**: per-SOP-Class mandatory modules,
    Type 1/2 presence, value multiplicity, enumerated values — top-level **and
    nested** per sequence item — plus **cross-reference integrity &amp; identifier
    uniqueness** (RT ROI / beam / setup references) and **conditional-presence
    rules** (palette / modality LUT / VOI window / lossy / pixel-padding). `--json`
    emits a structured findings document an agent can branch on:

    ```console
    $ dcmclient dcmvalidate ct.dcm --json            # findings as JSON
    $ dcmclient dcmvalidate study/*.dcm --errors-only # only error-severity
    ```

## Worked recipes

Real chains — each step is one tool call, so an agent can run them too.

**PACS → de-identified NIfTI for a model**

```console
$ dcmclient search https://pacs.example.com --patient-id 42 --modality CT --json
$ dcmclient pull   https://pacs.example.com --study 1.2.840... -o ./study
$ dcmclient dcmdeident ./study/*.dcm --out-dir ./deid     # PS3.15 de-identification
$ dcmclient dcm2nii ./deid ct.nii.gz                       # 3-D volume, validated affine
```

**Migrate a study between two PACSes (streaming, constant memory)**

```console
$ dcmclient forward --source https://old-pacs.example.com \
                    --dest   https://new-pacs.example.com \
                    --study 1.2.840...
```

**Bridge a legacy DIMSE sender to a DICOMweb archive**

```console
$ dcmclient listen --port 11112 --ae-title BRIDGE \
                   --sink https://archive.example.com    # C-STORE in → STOW out
```

**Inspect, fix a wrong tag, and re-validate — the LLM edit loop**

```console
$ dcmclient dcmdump ct.dcm | grep PatientID                # see the current value
$ dcmclient dcmodify ct.dcm -i '(0010,0020)=ANON-001'      # byte-verbatim edit
$ dcmclient dcmvalidate ct.dcm                             # confirm still conformant
```

**Author a structured report from measurements**

```console
$ dcmclient mkreport findings.json report.dcm              # TID 1500 SR
$ dcmclient dsr2html report.dcm > report.html              # human-readable render
```

## Agent / MCP

The same ToolSpec registry that backs the CLI is the agent surface:

```console
$ dcmclient manifest          # JSON self-description of every tool (inputSchema + outputSchema)
$ dcmclient mcp               # live MCP server over stdio — an agent drives the whole roster
```

Because the [tool reference](reference.md) is generated from `manifest`, the
docs, the agent surface, and the binary can never disagree.

## Exit codes

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | runtime error (bad input, or a per-file failure) |
| 104 | argument conversion error (bad numeric / enum) |
| 105 | validation error (rejected or mutually-exclusive option) |
| 106 | required option or argument missing |
