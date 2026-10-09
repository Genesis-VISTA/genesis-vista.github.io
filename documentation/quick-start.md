# Quick Start Guide

Use this guide to install VISTA, complete the first-launch setup, and begin working with the scientific assistant.

## 1. Install VISTA

Choose the command for your operating system.

### macOS

VISTA supports Apple Silicon Macs. Open Terminal and run:

```bash
curl -fsSL https://github.com/Genesis-VISTA/vista/releases/latest/download/install.sh | bash
```

### Linux

VISTA supports x86-64 Linux. Open a terminal and run:

```bash
curl -fsSL https://github.com/Genesis-VISTA/vista/releases/latest/download/install.sh | bash
```

### Windows

VISTA supports x64 Windows. Open PowerShell and run:

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://github.com/Genesis-VISTA/vista/releases/latest/download/install.ps1 | iex"
```

The installer verifies the downloaded package, installs VISTA, and opens the application.

!!! note "VISTA is a desktop application"

    VISTA requires a graphical desktop session and does not run over SSH or without a display. Linux installations also require `/dev/kvm`.

## 2. Configure the agent

On first launch:

1. Open **Settings** at the bottom of the sidebar.
2. Select **Agent**.
3. Paste your inference API key.
4. Return to Chat and choose an available model from the model selector.

Treat API keys as secrets. Do not place them in chats, documentation, screenshots, or source control.

## 3. Connect HPC clusters if needed

HPC configuration is optional. If you plan to submit jobs to a supported cluster, open **Settings** and connect each cluster using your own credentials.

You can skip this step if you only need chat, literature search, knowledge bases, or local analysis workflows.

## 4. Start working

VISTA is now ready to use. Create or open a project, select the appropriate skills and knowledge bases, and begin a conversation with the scientific agent.

[Continue to **How to use VISTA**](index.md){ .md-button .md-button--primary }

## Open VISTA again

After installation, launch VISTA like any other desktop application:

- **macOS:** Open `VISTA.app`.
- **Linux:** Open VISTA from the application menu.
- **Windows:** Open VISTA from the Start menu.

## Upgrade VISTA

Close VISTA and run the installer command for your operating system again. The installer upgrades the application while preserving its existing state.

For release packages and updates, visit the [VISTA releases page](https://github.com/Genesis-VISTA/vista/releases).

## Run from source

Developers who need to run VISTA from a repository checkout should follow the [source-development instructions](https://github.com/Genesis-VISTA/vista#run-from-source). Running from source requires Node.js 20.9 or later, `uv`, Docker or Podman, and access to the private dependencies used for HPC job submission.
