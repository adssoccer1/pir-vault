# pir-vault

Central repository for Post-Incident Review (PIR) markdown files.

A Cursor Automation watches the `main` branch of this repo. When a new PIR file is merged into `/pirs/`, the automation:

1. Reads the new PIR file
2. Validates its schema (see [`/pirs/README.md`](./pirs/README.md))
3. POSTs the parsed PIR to the local `pir-orchestrator` endpoint exposed via ngrok

The orchestrator then propagates the lesson across the org's services automatically — scanning candidates for the same anti-pattern and opening draft remediation PRs.

## Adding a new PIR

Create a new file in `/pirs/` named `YYYY-MM-DD-short-slug.md`. See [`/pirs/README.md`](./pirs/README.md) for the required schema.
