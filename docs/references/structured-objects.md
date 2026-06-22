# Structured objects — SEG, paramap, RT, SR, KOS

Read this when **reading or authoring** a semantic DICOM object (segmentation,
parametric map, RT plan/dose, structured report, key-object selection). These
are the objects an AI pipeline produces and sends back to a PACS.

## Read / inspect

| Object | Read it with |
|---|---|
| SEG / RT / PS / SR (auto-detect) | `dcm2content` |
| RT contours / ROIs | `dcm2content --contours rtstruct.dcm` |
| SR document tree | `dsrdump` · `dsr2html` (HTML) · `dsr2xml` (XML) |

## Author / write

| Object | Write it with |
|---|---|
| Coded segmentation (SEG) | `mkseg` |
| Parametric map | `mkparamap` |
| Structured report | `mksr` · `xml2dsr` (from edited XML) |
| Radiology report / KOS | `mkreport` · `mkkos` |

## Recipes

### Model labelmap → DICOM SEG → validate → export
```console
dcmclient mkseg ct.dcm --mask labelmap.dcm --meta seg.json -o out-seg.dcm
dcmclient dcmvalidate out-seg.dcm           # IOD-conformance gate before sending
dcmclient dcm2content out-seg.dcm           # confirm segments / coded concepts
dcmclient seg2nii  out-seg.dcm seg.nii.gz --masks   # round-trip to labelmap
```
`--meta seg.json` carries the coded anatomy / category / type. `--mask-raw
labelmap.u16` accepts a raw labelmap instead of a DICOM mask.

### RT dose review
```console
dcmclient dcm2content --contours plan-rtstruct.dcm        # list ROIs
dcmclient dcmdvh rtstruct.dcm rtdose.dcm --roi 1          # DVH for one ROI
dcmclient dcmdvh rtstruct.dcm rtdose.dcm --all dvh.json   # DVH for every ROI
```

### Structured report round-trip
```console
dcmclient dsrdump report.dcm               # human-readable tree
dcmclient dsr2xml report.dcm > rpt.xml     # → XML, edit, then:
dcmclient xml2dsr rpt.xml authored-sr.dcm  # XML → SR
```

## Guardrail

**Always `dcmvalidate`** an authored SEG / RT / SR / paramap before storing it
back — a malformed IOD can be silently rejected or corrupt the study.
