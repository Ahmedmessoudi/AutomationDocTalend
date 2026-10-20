# TalendDocs - Docs Migration (Batch Processing)

**Type:** Batch / on-demand migration
**Trigger:** Manual (`Execute workflow`)
**n8n export file:** [`workflow.json`](./workflow.json)

## Purpose

Bulk-generates (or regenerates) Confluence documentation for an entire folder of existing Talend job pages in one run. It's designed for a one-off or occasional migration: point it at a source Confluence folder containing job pages, and a target folder to publish into, and it will match each job to its corresponding `.item` file on GitHub, extract its structure, generate an AI sequence diagram, and create/update the documentation page for every job — without needing an event trigger for each one.

## How it works

1. **Trigger** — Started manually (`When clicking 'Execute workflow'`).
2. **Configuration** — The `Config` node hardcodes the run parameters (source/target Confluence folder IDs, GitHub repo/branch, space key — see [Configuration](#configuration-checklist) below).
3. **Job discovery** — `Confluence: Get Child Job Pages` lists all pages under the source folder to get the list of job names; `GitHub: Get Repo Tree` retrieves the full file tree of the repository.
4. **Matching** — `Extract & Match Job Files` pairs each job name to its most recent matching `.item` file in the repo tree. If no match is found, the job is logged and skipped (`Log Skipped Job (No Match)` → `Write Skipped Job Log`).
5. **Extraction** — For each matched job, the `.item` file is downloaded from GitHub and parsed (`XML`, `Code in JavaScript` nodes) to extract mappings, SQL, inputs and outputs.
6. **AI diagram generation** — The abstracted job structure is sent to an LLM (`Groq Chat Model` / `Basic LLM Chain`, with `OpenRouter Chat Model` and `Qwen Cloud Chat Model` available as alternatives) to produce a Mermaid sequence diagram of the data flow. The model's raw output is cleaned (`Extract Diagram From Chain`) and turned into an HTML documentation page (`HTML generation`).
7. **Smart publishing (upsert)** — For each job, the workflow checks the internal tracking table (`Get row(s)` / `If`) to see whether a Confluence page already exists in the **target** folder:
   - **Exists** → update the page (PUT, version + 1) and update the tracking row.
   - **New job** → create the page (POST) in the target folder and insert a new tracking row.

## Data table: `MigrationTable`

Created once via the `Create a data table` node before the first run.

| Column | Type | Description |
|---|---|---|
| `jobName` | string | Talend job name |
| `confluencePageId` | number | ID of the corresponding Confluence page (in the target folder) |
| `confluenceVersion` | number | Current version number of the Confluence page |

> ⚠️ Run the `Create a data table` node **once** before the workflow's first execution. Don't re-run it afterwards (it will wipe the table).

## Required credentials

| Service | Credential type | Notes |
|---|---|---|
| Confluence | Basic Auth | Username = Atlassian account email, Password = Confluence API token |
| GitHub | GitHub OAuth2 / Personal Access Token | Used to read the repo tree and file contents |
| Groq (LLM) | Groq API Key | Used by `Groq Chat Model` (default) |
| OpenRouter (optional fallback LLM) | OpenRouter API Key | Used by `OpenRouter Chat Model` |
| Qwen / Alibaba Cloud (optional fallback LLM) | Alibaba Cloud API Key | Used by `Qwen Cloud Chat Model` |

## Configuration checklist

Before running this workflow, set the following values in the **`Config`** node:

```json
{
  "sourceFolderId": 9240606,
  "targetFolderId": 9437211,
  "confluenceBaseUrl": "https://<your-domain>.atlassian.net/wiki",
  "branch": "main",
  "githubOwner": "integration-team-OS",
  "githubRepo": "AutomatisationDocTalend",
  "spacekey": "Test"
}
```

- [ ] `sourceFolderId` — Confluence folder ID containing the existing job pages to migrate
- [ ] `targetFolderId` — Confluence folder ID to publish the generated documentation into
- [ ] `confluenceBaseUrl`, `spacekey` — Target Confluence instance/space
- [ ] `githubOwner`, `githubRepo`, `branch` — Repository containing the Talend `.item` files
- [ ] Confluence, GitHub and LLM credentials attached to their respective nodes
- [ ] `MigrationTable` created (run `Create a data table` once)
- [ ] LLM model choice finalized (see in-workflow TO-DO sticky note)

## Notes

- New pages are tagged **`[refactored]`** by default. This label can be changed in the `HTML generation` node (around line 314 of its code).
- This workflow is intended to be run **manually**, typically once per migration batch — not left active/listening.
