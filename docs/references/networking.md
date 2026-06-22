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
dcmclient wado   --server $URL --study 1.2.840.. -o ./study           # pull
```

### DIMSE retrieval loop
```console
dcmclient echo-scu --aet ME --aec PACS --host pacs --port 104          # verify
dcmclient find-scu --aec PACS --host pacs --port 104 --study-uid 1.2.. # query
dcmclient get-scu  --aec PACS --host pacs --port 104 --study-uid 1.2.. -o ./study
```

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
