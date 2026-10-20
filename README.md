# AutomatisationDocTalend — n8n Workflows

Automated documentation pipeline that keeps Confluence pages for Talend ETL jobs in sync with what is actually deployed. Talend job structures (mappings, SQL, inputs/outputs) are extracted from GitHub, turned into an AI-generated sequence diagram of the data flow, and published to Confluence — either automatically on every GitHub push, or in bulk for an initial/occasional migration.

## Overview

This repository contains two n8n workflows that cover the two ways documentation gets generated:

| Workflow | Folder | Trigger | Use case |
|---|---|---|---|
| **TalendDocs-Tags** | [`/TalendDoc`](./TalendDo) | GitHub push (webhook) | Continuous, event-driven sync — keeps Confluence up to date automatically every time a job is pushed |
| **TalendDocs - Docs Migration (Batch)** | [`/TalendDocsMigration`](./TalendDocsMigration) | Manual | One-off / occasional bulk migration of an entire folder of existing job pages |

Each folder contains:
- `README.md` — full documentation of that workflow (how it works, credentials, configuration)
- `workflow.json` — the n8n workflow export, ready to import

## Architecture (shared pipeline)

Both workflows follow the same core pipeline once a job is selected for (re)documentation:

```
GitHub (.item file)
      │
      ▼
Parse XML (mappings, SQL, inputs/outputs)
      │
      ▼
LLM (Groq / OpenRouter / Qwen) → Mermaid sequence diagram
      │
      ▼
Render HTML documentation page
      │
      ▼
Confluence — create page (new job) OR update page (existing job)
      │
      ▼
Internal tracking table (data table) — stores jobName ↔ Confluence pageId ↔ version
```

The two workflows differ mainly in **what triggers them** and **how they select which jobs to process**:
- `TalendDocs-Tags` reacts to a single GitHub push and checks the Talend Management Console (TMC) for a version bump before regenerating docs for that job.
- `TalendDocs - Docs Migration` is run manually against a whole Confluence source folder, matches every job page to its GitHub file, and republishes them all into a target folder.

## Deployment / Setup

### 1. Prerequisites

- An n8n instance (self-hosted or cloud) with the following community/base nodes available: `httpRequest`, `dataTable`, `code`, `xml`, `readWriteFile`, `githubTrigger`, `@n8n/n8n-nodes-langchain` (LLM chain nodes).
- A GitHub repository containing the Talend `.item` source files.
- A Confluence Cloud space to publish documentation into.
- A Talend Management Console (TMC) account (for `TalendDocs-Tags` version checking).
- An API key for at least one supported LLM provider (Groq, OpenRouter, or Alibaba/Qwen).

### 2. Import the workflows

In n8n: **Workflows → Import from File**, and import each `workflow.json` from the two folders above.

### 3. Configure credentials

Create the following credentials in n8n and attach them to the relevant nodes (see each workflow's README for the exact nodes):

| Credential | Type | Used by |
|---|---|---|
| GitHub | OAuth2 / Personal Access Token | Both workflows |
| Confluence | Basic Auth (email + Confluence API token) | Both workflows |
| Talend TMC | Header Auth (`Authorization: Bearer <API key>`) | `TalendDocs-Tags` |
| Groq | API Key | Both workflows (default LLM) |
| OpenRouter *(optional)* | API Key | `TalendDocs - Docs Migration` (fallback LLM) |
| Alibaba Cloud / Qwen *(optional)* | API Key | Both workflows (fallback LLM) |

### 4. Configure workflow parameters

- **`TalendDocs-Tags`**: set the GitHub owner/repo/branch on the `Github Trigger` node, and the Confluence base URL / space key on the `update page`, `add page`, and `get last version` nodes.
- **`TalendDocs - Docs Migration`**: set `sourceFolderId`, `targetFolderId`, `confluenceBaseUrl`, `branch`, `githubOwner`, `githubRepo`, and `spacekey` in the **`Config`** node.

### 5. Initialize the tracking tables

Each workflow keeps its own n8n Data Table to track which jobs are already documented (`jobName` ↔ Confluence `pageId` ↔ version). Run each workflow's `Create a data table` node **once** before the first real execution:

- `TalendDocs-Tags` → creates `Version Table`
- `TalendDocs - Docs Migration` → creates `MigrationTable`

### 6. Activate / run

- `TalendDocs-Tags` → set the workflow **Active** in n8n so the GitHub webhook trigger listens for pushes.
- `TalendDocs - Docs Migration` → run manually whenever a bulk (re)generation is needed; leave inactive otherwise.


## Notes for the maintainer

- LLM model choice is still open for tuning — see the in-workflow sticky notes ("Make a decision on what model to use", "change the models for better performance").
- New Confluence pages created by the migration workflow are tagged `[refactored]` by default (configurable in its `HTML generation` node).
- Both workflows are currently exported as **inactive**; review credentials and configuration before activating.
