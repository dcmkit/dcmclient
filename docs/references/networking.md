# Networking — query, retrieve, store, route

Read this when the task involves a **PACS / DICOMweb server** (search, pull,
push, route, receive) or DIMSE-N workflow messaging. Exact flags:
`dcmclient <tool> --help`.

## Two transports, same tasks

| Task | DICOMweb | DIMSE |
|---|---|---|
| Query | `search` (QIDO-RS, alias `qido`) | `find-scu` (C-FIND) |
| Retrieve | `wado` (WADO-RS, alias `pull`) · `wado-uri` | `get-scu` (C-GET) · `move-scu` (C-MOVE) |
| Store | `stow` (STOW-RS, alias `push`) | `store-scu` (C-STORE) |
| Delete | `delete` | — |
| Receive | `listen` | `store-scp` |
| Bridge | `forward` (web ↔ DIMSE in either direction) | |
| Verify | — | `echo-scu` (C-ECHO) |
| Workflow / status | — | `mpps-scu` · `ups-scu`/`ups` · `stgcmt-scu` · `ian-scu` · `print-scu` · `term-scu` |

## Endpoints & auth

- **DICOMweb** always needs `--server <base-url>`. Auth, when required:
  - AWS SigV4: `--aws-region` (+ `--aws-access-key` / `--aws-secret-key` /
    `--aws-session-token`, or the standard `AWS_*` env vars). Enables signing for
    AWS HealthImaging-style endpoints.
  - OAuth2: `--oauth2-token-url` / `--oauth2-client-id` / `--oauth2-client-secret`
    / `--oauth2-scope` (client-credentials, refresh, loopback, or device flow;
    `--oauth2-flow` to force one). Bearer/Basic also accepted by the client.
- **DIMSE** needs the AE triple: calling AE, called AE, host, port (see
  `--help`). TLS variants exist where built.

## Recipes

### DICOMweb retrieval loop
```console
dcmclient search --server $URL --patient-id 42 --modality CT --json   # find
dcmclient wado   --server $URL --study 1.2.840.. -o ./study           # pull (server default encoding)
dcmclient wado   --server $URL --study 1.2.840.. --transfer-syntax '*' -o ./study   # pull verbatim
```

`--transfer-syntax` sets the WADO-RS `transfer-syntax` Accept parameter: omitted = the server's
default (PS3.18: Explicit VR Little Endian — possibly transcoded); `'*'` = the stored encoding
verbatim (no transcoding, e.g. keep JPEG-2000 as-is); a TS UID = that encoding if the server can
produce it.

### DIMSE retrieval loop

Host and port are positional; matching keys are `-k Key=Value` (same as `find-scu`).

```console
dcmclient echo-scu pacs 104 --aet ME --aec PACS                          # verify
dcmclient find-scu pacs 104 --aec PACS -S -k StudyInstanceUID=1.2..      # query
dcmclient get-scu  pacs 104 --aec PACS -S -k StudyInstanceUID=1.2.. \
                   --store-class 1.2.840.10008.5.1.4.1.1.2 --out-dir ./study   # retrieve
```

**Declare the Storage classes you'll receive.** C-GET streams matches back as C-STORE
sub-operations on the same association, so each class needs a negotiated context — and a
non-transcoding server aborts the whole retrieval if a matched instance has none. Pass one
repeatable `--store-class <SOP Class UID>` per class the `find-scu` showed (they then negotiate
their full transfer-syntax set, so compressed instances come back and are written verbatim).
Omit `--store-class` and `get-scu` offers the common image and report classes.

### Bridge web ↔ DIMSE
```console
dcmclient forward --server $URL --study 1.2.. --dest-aec PACS --dest-host pacs --dest-port 104
```

## Guardrails

- **De-identify before egress.** Any `stow` / `forward` / `store-scu` to a remote
  node sends patient data off-box — run `dcmdeident` first.
- Network tools surface the DICOM response status inside the `--json` `result` —
  branch on it, not on stderr text.
- Never assume a default endpoint; require an explicit `--server` / AE triple.
