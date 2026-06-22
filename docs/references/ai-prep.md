# AI prep — volumes, NIfTI, raw pixels, resampling, radiomics

Read this when the task is **feeding imaging into a model** or computing
quantitative features. Always `dcmdeident` first if the data left a PACS.

## Tools

| Need | Tool |
|---|---|
| 3-D volume → NIfTI (+ BIDS sidecar) | `dcm2nii` |
| Raw decoded pixels (native ints, HU) for NumPy/PyTorch | `dcm2raw` |
| Resample to a target grid | `dcmresample` |
| Radiomics features → JSON | `dcmradiomics` |
| SEG mask → labelmap NIfTI | `seg2nii` |

## Recipes

### Series → model input
```console
dcmclient dcmdeident ./study/*.dcm --out-dir ./deid    # de-id first
dcmclient dcm2nii    ./deid ct.nii.gz                  # 3-D volume, geometry-correct
# dcm2nii also emits a BIDS .json sidecar where applicable (--bids)
```

### Raw pixels (skip NIfTI, go straight to an array)
```console
dcmclient dcm2raw --json scan.dcm pixels.raw   # result has dtype/shape/rescale
```
Pixels come out as native integers; pass the rescale slope/intercept (in the
`--json` result) to get HU/SUV.

### Resample to an isotropic grid
```console
dcmclient dcmresample series-dir/ out.nii.gz --spacing 1,1,1   # mm
# or --size W,H,D for an explicit voxel grid
```

### Radiomics over an ROI
```console
dcmclient dcmradiomics ct.dcm --mask roi.dcm --json
```

## Notes

- `dcm2nii` keeps the true (possibly sheared) affine; orthogonalize on demand via
  `dcmresample` rather than expecting deskew as a separate step.
- Geometry is single-sourced from the volume model matrix — don't hand-compute
  affines from IPP/IOP.
