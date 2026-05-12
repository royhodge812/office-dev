Managing a webapp on a DigitalOcean Droplet via OpenClaw can lead to "ghost code" and drift if the LLM is writing directly to the live environment without a Git-backed source of truth. If the VM cycles or the LLM loses its state, your friend is left with a headache.

Here is a hardened Daily Backup & Weekly Restoration plan designed to anchor the LLM and protect the source code.

## 1. The Strategy: "Git as the Anchor"

The primary reason LLMs "lose" code is that they treat the filesystem as a scratchpad. You must force a Git-first workflow.

*   **Source of Truth:** All code lives in a Private GitHub/GitLab repo.
*   **The Droplet:** Acts only as a runner.
*   **The LLM:** Must be configured to git commit and push after every successful change.

### Daily Backup Plan (Automated)

We need to back up two things: the Database/State (OpenClaw's memory) and the Filesystem.

| Component | Tool | Frequency | Destination |
| :--- | :--- | :--- | :--- |
| Source Code | Git (GitHub/GitLab) | Per Change | Remote Repository |
| OpenClaw State | openclaw backup | Every 6 Hours | DigitalOcean Spaces (S3) |
| Full VM Image | DO Snapshots | Daily | DigitalOcean Infrastructure |

### The "No-Headache" Script:

Create a cron job on the Droplet (`/etc/cron.daily/backup-openclaw`) to ship the state off-box:

```bash
#!/bin/bash
# Backup OpenClaw state & workspace
DATE=$(date +%Y%m%d)
BACKUP_NAME="oc-backup-$DATE.tar.gz"

# Generate internal OpenClaw backup
openclaw backup --output /tmp/$BACKUP_NAME

# Sync to S3-compatible storage (DigitalOcean Spaces)
# rclone copy /tmp/$BACKUP_NAME remote:my-backups/openclaw/
rm /tmp/$BACKUP_NAME
```

## 2. Weekly Restoration Drill (The Safety Net)

A backup is a hallucination until you've restored it. Every Sunday, your friend should perform a "Shadow Restore" to a temporary Droplet to ensure the LLM hasn't corrupted the state.

1.  Spin up a new Droplet from the latest DO Snapshot.
2.  Pull the latest code via `git pull` to ensure the workspace is identical.
3.  Run `openclaw restore` using the latest `.tar.gz` from your Spaces/S3 bucket.
4.  **Verify the LLM "Context":** Ask the LLM on the new VM: "What was the last feature we worked on yesterday?" If it knows, the memory restoration worked.

## 3. Preventing "LLM Headaches" (Operational Hardening)

To stop the LLM from hallucinating its way into a broken app, implement these Guardrails:

### A. Environment Separation
Stop the LLM from coding on the production VM.
*   **Dev Droplet:** Where the LLM lives and breaks things.
*   **Prod Droplet:** Code is only deployed here via a git push or a CI/CD trigger after the friend reviews it.

### B. The "Checkpoint" System
Teach your friend to use Atomic Commits. Before the LLM starts a new task, it should run:
`git checkout -b task-description`
If the LLM starts looping or making mistakes, your friend can simply `git checkout main` and delete the "hallucinated" branch.

### C. Persistent Memory Tooling
OpenClaw often loses track if the session tokens or local SQLite DBs get wiped.
*   **Hard-link the workspace:** Ensure the OpenClaw workspace directory is a dedicated Git repo.
*   **API Key Safety:** Store all DigitalOcean and LLM keys in a `.env` file that is in `.gitignore`. Never let the LLM "manage" the keys; keep them as environment variables on the Droplet.

## Implementation Checklist

- [ ] **Enable DigitalOcean Backups:** (20% extra cost, but worth it for the 1-click restore).
- [ ] **Setup rclone:** To move OpenClaw .tar.gz files to off-site storage.
- [ ] **Force Git usage:** If the LLM doesn't commit, it didn't happen.
- [ ] **Snapshot before "Big Moves":** Before letting the LLM do a major refactor, take a manual DigitalOcean Snapshot.
