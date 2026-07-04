# Post-Setup Assertions

The first boot has one unavoidable manual step: a human must run `codex login --device-auth`. The generated setup therefore separates assertions into two groups.

## Automatic Pre-Auth Assertions

These run in `codex-agent-setup.service` before Codex auth:

- `node --version`, `npm --version`, and `npx --version` work from the systemd-launched setup context.
- `codex --version` works.
- `~/.codex/config.toml` exists for the generated agent user.
- Codex config contains:
  - `sandbox_mode = "danger-full-access"`
  - `approval_policy = "never"`
  - `web_search = "live"`
  - trusted project entry for the agent home directory
- Bootstrap skills are installed:
  - `manage-durable-notes`
- Passwordless sudo works for the generated agent user.
- SSH is active.
- Avahi is active.
- Effective `sshd -T` contains public-key auth on, root login off, password auth at the configured first-access setting, and the generated user in `allowusers`.
- Durable notes and remote access notes exist.

Results are written to:

```text
/var/log/codex-agent-health-summary.log
```

If any automatic assertion fails, the setup service exits nonzero and leaves logs in place for inspection.

## Manual Post-Auth Smoke Test

After SSH login and `codex login --device-auth`, start a fresh Codex session and run a minimal authenticated smoke test:

```bash
codex --version
codex exec --skip-git-repo-check 'Reply with OK if Codex auth is working.'
```

This is intentionally not part of first boot because auth cannot be completed unattended.
