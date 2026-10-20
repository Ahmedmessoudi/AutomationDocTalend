# TalendDocs-Tags

**Type:** Event-driven (real-time sync)
**Trigger:** GitHub push webhook
**n8n export file:** [`workflow.json`](./workflow.json)

## Purpose

Keeps Confluence documentation for Talend jobs automatically in sync with what is actually deployed. Every time a Talend job is pushed to GitHub, the workflow checks whether the deployed artifact version (from Talend Management Console) is newer than the version already documented. If it is, it re-extracts the job's structure, regenerates an AI-produced sequence diagram of the data flow, and creates or updates the corresponding Confluence page — with no manual step required.

## How it works

1. **Trigger** — `Github Trigger` node listens for `push` events on the configured repository/branch.
2. **Version check** — The workflow calls the Talend Management Console (TMC) API (`TMC`, `get last version` nodes) to fetch the latest deployed artifact version for each job, and compares it against the version stored in the internal tracking table (`Version Table`). If nothing changed, the job is skipped.
3. **Extraction** — The updated `.item` file is downloaded from GitHub and parsed (`XML`, `mapper` nodes) to extract mappings, SQL statements, inputs and outputs.
4. **AI diagram generation** — The abstracted job structure is sent to an LLM (`Groq Chat Model` / `Basic LLM Chain`, with `Qwen Cloud Chat Model` as an alternative) which returns a Mermaid sequence diagram of the job's data flow. The raw model output is cleaned up (`Extract Diagram From Chain`) and rendered into an HTML documentation page (`HTML generator`).
5. **Smart publishing (upsert)** — The workflow checks (`add || update` node) whether a Confluence page already exists for the job (via the tracking table):
   - **Exists** → `update page` (PUT, version + 1) and the tracking row is updated.
   - **New job** → `add page` (POST) and a new row is inserted into `Version Table`.
6. **Logging** — Every run is logged (`update logs`, `add logs`) for traceability.

## Data table: `Version Table`

Created once via the `Create a data table` node before the first run.

| Column | Type | Description |
|---|---|---|
| `jobName` | string | Talend job name |
| `talendVersion` | string | Last deployed artifact version (from TMC) |
| `confluencePageId` | number | ID of the corresponding Confluence page |
| `confluenceVersion` | number | Current version number of the Confluence page |

> ⚠️ Run the `Create a data table` node **once** before the workflow's first execution. Don't run it again afterwards (it will wipe the table).

## Required credentials

| Service | Credential type | Notes |
|---|---|---|
| GitHub | GitHub OAuth2 / Personal Access Token | Used by the `Github Trigger` node; requires repo webhook permissions |
| Talend TMC | Header Auth | `Authorization` header = `Bearer <TMC API Key>` |
| Confluence | Basic Auth | Username = Atlassian account email, Password = Confluence API token |
| Groq (LLM) | Groq API Key | Used by `Groq Chat Model` |
| Qwen / Alibaba Cloud (optional fallback LLM) | Alibaba Cloud API Key | Used by `Qwen Cloud Chat Model` |

## Configuration checklist

Before activating this workflow, update the following in the relevant nodes:

- [ ] GitHub owner / repository / branch (`Github Trigger` node)
- [ ] Confluence base URL, space key (`update page` / `add page` / `get last version` nodes)
- [ ] TMC base URL (`TMC` node)
- [ ] Confluence, GitHub, TMC and LLM credentials attached to their respective nodes
- [ ] `Version Table` created (run `Create a data table` once)
- [ ] LLM model choice reviewed for cost/performance trade-offs (see in-workflow sticky note)

## Notes

- Confluence credentials use **Basic Auth**: email + Confluence API key.
- TMC credentials use **Header Auth**: `Authorization` = `Bearer <API key>`.
- The workflow is currently deployed as **inactive**; activate it in n8n once credentials and configuration are set.
