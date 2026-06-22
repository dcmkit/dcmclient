# Agent integration

`dcmclient` is built to be driven by LLM agents. There are two entry points —
a live **MCP server** for MCP-aware runtimes, and a **`manifest`** for any
framework that wants raw JSON tool schemas. Both come from the same internal
registry, so they always match the binary exactly.

## MCP server

`dcmclient mcp` speaks the [Model Context Protocol](https://modelcontextprotocol.io)
over stdio (JSON-RPC: `tools/list` + `tools/call`). Every one of the 61 tools
is exposed automatically.

### Claude Desktop / Claude Code

Add the server to your MCP client config. For a local binary:

```json
{
  "mcpServers": {
    "dcmclient": {
      "command": "/usr/local/bin/dcmclient",
      "args": ["mcp"]
    }
  }
}
```

Use the absolute path to the binary (or just `dcmclient` if it's on the
client's `PATH`).

For a sandboxed runtime that can't install a binary, run the container instead
(mount the data you want reachable):

```json
{
  "mcpServers": {
    "dcmclient": {
      "command": "docker",
      "args": ["run", "--rm", "-i",
               "-v", "/path/to/dicom:/work",
               "ghcr.io/dcmkit/dcmclient", "mcp"]
    }
  }
}
```

Restart the client; all 61 tools appear, callable by name with schema-checked
arguments. `dcmclient mcp -v` logs JSON-RPC activity to stderr for debugging.

## manifest — raw tool schemas

For a framework that builds its own tool definitions (OpenAI function calling,
LangChain, a custom agent loop), `dcmclient manifest` emits the whole registry
as JSON:

```console
$ dcmclient manifest
```

```json
{
  "manifestVersion": "...",
  "tools": [
    {
      "name": "dcm2raw",
      "purpose": "Decode DICOM pixels (any transfer syntax) to raw native integers ...",
      "category": "render",
      "keywords": ["raw", "pixels", "decode", "numpy", "ai", "hu", "ml"],
      "readOnly": false, "writesFiles": true, "network": false,
      "examples": ["dcmclient dcm2raw --json scan.dcm pixels.raw"],
      "inputSchema":  { "...": "JSON Schema for the arguments" },
      "outputSchema": { "...": "JSON Schema for the --json result" }
    }
  ],
  "exitCodes": { "0": "success", "1": "runtime error", "...": "..." },
  "conventions": "stdout = machine output, stderr = human diagnostics ..."
}
```

Each tool's `inputSchema` / `outputSchema` are standard JSON Schema — map them
straight onto your framework's tool/function type. Filter by `readOnly`,
`network`, or `writesFiles` to expose only a safe subset to an agent.

## Calling tools: the contract

Whether via MCP or by shelling out, every tool follows the same conventions:

- **`stdout` is machine output, `stderr` is human diagnostics.** Status tools
  (the DIMSE SCUs/SCPs, `dcmprobe`, `dcmconv`, `json2dcm`, `xml2dcm`) take
  `--json` and return a `{tool, ok, exit_code, result, error}` envelope.
  Document tools (`dcmdump`, `dcm2json`, `dcm2xml`, `dsrdump`, `dsr2html`,
  `dcmvalidate`) write their structured artifact straight to stdout.
- **Network tools** also surface the DICOM response status inside `result`.
- **Exit codes** are graduated, so an agent can branch on the failure class:

| code | meaning |
|---|---|
| 0 | success |
| 1 | runtime error — unreadable / invalid input, or a per-operation failure |
| 104 | bad numeric or enum argument value |
| 105 | rejected value, or mutually-exclusive options |
| 106 | required option or argument missing |

## End-to-end: an agent triage run

A representative chain an agent can execute, each step a single tool call:

```console
# 1. find the study on the PACS
$ dcmclient search --server https://pacs.example.com --patient-id 42 --modality CT --json

# 2. pull it locally
$ dcmclient pull --server https://pacs.example.com --study 1.2.840... -o ./study

# 3. de-identify before any downstream processing
$ dcmclient dcmdeident ./study/*.dcm --out-dir ./deid

# 4. build a 3-D volume / NIfTI for a model
$ dcmclient dcm2nii ./deid ct.nii.gz

# 5. compute radiomic features over an ROI → JSON
$ dcmclient dcmradiomics ct.dcm --mask roi.dcm --json
```

Under MCP the agent issues the same five calls as `tools/call` requests,
reading each JSON result to decide the next step — the schemas and the
envelope give it everything it needs to chain them without bespoke glue.
