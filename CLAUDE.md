# CLAUDE.md — Homelab.Stacks.BuildLab

Guidance for Claude Code working in this repo.

## What this is

A **stack submodule** of [`Chrison-Homelab/Homelab`](https://github.com/Chrison-Homelab/Homelab),
mounted there at `stacks/BuildLab` (meta-repo model, ADR-0008). This repo holds the stack's shapes
plus the Windows provisioning assets; the **converge/validate engine lives in the superproject**
and is the only thing that applies them.

> **Read the superproject's [`CLAUDE.md`](https://github.com/Chrison-Homelab/Homelab/blob/main/CLAUDE.md) first.**
> It carries the rules that apply here and are *not* repeated in this file: the PR-only git
> workflow + merge strategy (ADR-0010), the worktree rule for parallel sessions, the shared
> external-account guardrails, and `secrets.env` / Bitwarden Secrets Manager handling.

## This stack

**Windows dev/build VMs** for testing the Fallout build project across multiple Visual Studio
toolchains.

- **VMID block:** `1100–1199` (enforced by `stack.yaml`'s `ctidRange`)
- **Default node:** `desktop-01` — strongest CPU (Ryzen 5 3600) and the proven Win11 recipe
- **Current member:** `buildvm` — **VMID 1100**, `kind: VM`

**`kind: VM`, not LXC.** These are provisioned by the **ProxmoxSharp write path** (the same path
the Gaming stack uses), *not* the community-scripts `ct/<app>.sh` path that the LXC stacks use.
Members set `vmid` explicitly — omission is an error.

`spec.defaults` in `stack.yaml` reuses the LXC spec shape (per the schema), so only fields common
to both (`node`/`timezone`/`tags`) belong there. VM-specific fields (`machine`/`bios`/`cpu`) stay
in the member shape — same convention as the Gaming stack.

## Working here

Converge runs **from the superproject**, pointed at this directory:

```bash
# in the superproject
dotnet run --project Infrastructure/engine -- validate stacks/BuildLab
dotnet run --project Infrastructure/engine -- converge stacks/BuildLab          # dry run
dotnet run --project Infrastructure/engine -- converge stacks/BuildLab --apply
```

## Gotchas specific to this stack

Windows unattended provisioning is unforgiving; these all cost real debugging time:

- **The autounattend ISO must be built WITHOUT Joliet.** Joliet mangles long filenames, and the
  provisioner then can't find its own files.
- **Provisioner scripts must be written as UTF-8 without CP1252 smart characters.** An em-dash in a
  PowerShell provisioner silently corrupted execution. Keep ASCII in anything that ends up on the
  Windows guest.
- **VS 2026 uses channel `18/stable`, not `18/release`.** The wrong channel installs nothing while
  appearing to succeed.
- **Stale node artifacts win over new ones.** If a rebuild seems to have no effect, check for an
  older ISO/answer file still sitting on the node.
- **Guest configuration goes through `qm guest exec`** (the agent is on the VM), not by fetching
  raw URLs — this is a private repo, so `irm` of a raw GitHub URL will not work.

All six VS toolchains (2019/2022/2026 × Community + BuildTools) plus .NET 10 install unattended as
of the last verified build.
