# Run Claude Code interactively on GitHub Actions runners

> **Goal:** Launch a GitHub Actions job on a chosen runner, open an **interactive terminal session**, and run **Claude Code (`claude`)** against the checked-out repository.

## What this supports

* ✅ Run Claude Code **interactive mode** (`claude`) in a real TTY session ([Claude Code][2])
* ✅ Choose **runner types** (GitHub-hosted Linux/macOS/Windows) or **self-hosted** by label ([GitHub Docs][3])
* ✅ Work on **one repository** (the workflow checks out the repo)
* ⚠️ Intended for **debug/dev use**. This opens remote shell access to your runner.

---

## Security notes (read first)

1. **Only allow manual triggers** (`workflow_dispatch`), and restrict who can run workflows.
2. Prefer **self-hosted runners** in a locked-down network if you need more control.
3. Use a GitHub Secret for `ANTHROPIC_API_KEY`. Avoid pasting secrets into terminal history.
4. Treat the tmate session as **break-glass** access: it provides direct shell access to the runner. ([GitHub][1])

---

## Prereqs

### 1) Claude Code API key

Store your key in repo secrets:

* Repo → **Settings** → **Secrets and variables** → **Actions**
* Create secret: `ANTHROPIC_API_KEY`

Claude Code can be installed via installer scripts (Linux/macOS/WSL, Windows PowerShell/CMD). ([Claude Code][4])

### 2) (Optional) self-hosted runners with labels

If using self-hosted runners, make sure they’re registered and have labels you can target via `runs-on`. GitHub routes jobs based on matching labels/groups. ([GitHub Docs][3])

---

## Step 1 — Add the workflow

Create: `.github/workflows/claude-interactive.yml`

```yaml
name: Claude Code (Interactive)

on:
  workflow_dispatch:
    inputs:
      runner_profile:
        description: "Which runner profile to use?"
        required: true
        type: choice
        options:
          - ubuntu-latest
          - macos-latest
          - windows-latest
          # self-hosted examples (must match your runner labels)
          - self-hosted-linux-x64
          - self-hosted-linux-arm64

jobs:
  interactive:
    name: Interactive shell on ${{ inputs.runner_profile }}
    runs-on: ${{ inputs.runner_profile == 'self-hosted-linux-x64' && fromJSON('["self-hosted","linux","x64"]')
             || inputs.runner_profile == 'self-hosted-linux-arm64' && fromJSON('["self-hosted","linux","arm64"]')
             || inputs.runner_profile }}
    timeout-minutes: 60

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # Install Claude Code
      # For Linux/macOS runners:
      - name: Install Claude Code (bash)
        if: runner.os != 'Windows'
        run: |
          curl -fsSL https://claude.ai/install.sh | bash

      # For Windows runners:
      - name: Install Claude Code (PowerShell)
        if: runner.os == 'Windows'
        shell: pwsh
        run: |
          irm https://claude.ai/install.ps1 | iex

      # Start an interactive shell session on the runner.
      # Connect details will appear in the Actions logs.
      - name: Start tmate session
        uses: mxschmitt/action-tmate@v3
        with:
          limit-access-to-actor: true

      # NOTE:
      # After connecting via tmate, you will manually run:
      #   export ANTHROPIC_API_KEY="${{ secrets.ANTHROPIC_API_KEY }}"
      #   claude
```

**Why tmate?** It provides an interactive shell inside the Actions environment via SSH or web shell. ([GitHub][1])

---

## Step 2 — Run the workflow

1. Go to **Actions** tab → “Claude Code (Interactive)”
2. Click **Run workflow**
3. Choose a `runner_profile`:

   * `ubuntu-latest` / `macos-latest` / `windows-latest` (GitHub-hosted)
   * `self-hosted-*` (your own, if configured)

---

## Step 3 — Attach to the runner (tmate)

1. Open the workflow run
2. Find the step **“Start tmate session”**
3. In the logs, you’ll see connection instructions (SSH and/or web link).

This is the “real-time” terminal connection to the runner. ([GitHub][1])

---

## Step 4 — Start Claude Code interactive mode

Once attached to the runner shell:

### Linux/macOS

```bash
export ANTHROPIC_API_KEY="${ANTHROPIC_API_KEY:-}"
# If it's empty, set it explicitly (recommended):
export ANTHROPIC_API_KEY="***"
claude
```

### Windows (PowerShell)

```powershell
$env:ANTHROPIC_API_KEY="***"
claude
```

Claude Code interactive mode starts by running `claude`. ([Claude Code][2])

You’re now working against the repo checkout on that runner.

---

## Step 5 — Work on the repo

Inside the interactive Claude session you can do typical repo work:

* inspect files
* run tests/build
* edit code (depending on your environment/tools)
* create commits (optional)

**Note:** On GitHub-hosted runners, the environment is ephemeral. Any unpushed commits vanish when the job ends.

---

## Step 6 — End the session

* Exit Claude (`Ctrl+C` / `exit` depending on mode)
* Disconnect from tmate
* Cancel the workflow run (or let it time out)

---

# Runner strategy guide

## Use GitHub-hosted runners when…

* quick testing/debugging
* OS matrix checks (Linux/macOS/Windows)

## Use self-hosted runners when…

* you need access to internal networks
* you want persistent caches/tooling
* you need stricter isolation and auditing

GitHub matches `runs-on` to self-hosted runner labels/groups to route jobs. ([GitHub Docs][3])

---

# Troubleshooting

## “I can’t connect to tmate”

* Some orgs block outbound connections from runners or restrict Actions.
* Try a self-hosted runner in a network that allows outbound access, or use a different debug mechanism.

## Claude command not found

* Confirm install step ran and your shell PATH includes the installed binary.
* Re-run the install command from the attached shell using the setup docs. ([Claude Code][4])

## Key not detected / auth errors

* Confirm `ANTHROPIC_API_KEY` is set in the shell session.
* Prefer setting it as a repo secret and exporting it after you attach.

---

## References

* Claude Code install/setup scripts ([Claude Code][4])
* Claude Code interactive mode and CLI docs ([Claude Code][2])
* action-tmate (interactive shell in GitHub Actions) ([GitHub][1])
