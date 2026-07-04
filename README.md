# Codex Create Agent Boot Drive CLI

Create a Raspberry Pi OS Lite ARM64 USB boot drive that becomes a reachable, dedicated Codex agent host on first boot.

This repository is public. Do not commit Wi-Fi credentials, initial passwords, private keys, tokens, Codex auth files, or recovery codes. Put local values in an ignored `*.local.toml` file.

## Current Target

- Raspberry Pi OS Lite ARM64 Trixie
- Raspberry Pi 5 booting from USB storage
- First access over SSH password auth
- Latest official Node.js LTS installed system-wide for interactive shells and first-boot/systemd setup tasks
- Codex auth remains a manual human step with `codex login --device-auth`
- Codex YOLO defaults are written before auth, so the next Codex session starts with local full access

## Usage

Create a local config from the example:

```bash
cp examples/agent.local.example.toml ./agent.local.toml
chmod 600 ./agent.local.toml
```

Edit the local file with the agent name, local user, temporary password, Wi-Fi SSID, Wi-Fi PSK, and drive guardrails. The agent `name` is the durable agent identity and generated `.local` hostname; `user` is the Unix account used for SSH. Existing deployments often use the same value for both, but the fields are intentionally separate.

List candidate drives:

```bash
lsblk -o NAME,PATH,VENDOR,MODEL,SERIAL,SIZE,TYPE,TRAN,RM,FSTYPE,LABEL,MOUNTPOINTS
find /dev/disk/by-id -maxdepth 1 -type l -printf '%p -> %l\n' | sort
```

Create the boot drive:

```bash
./bin/create-agent-boot-drive \
  --destination /dev/disk/by-id/usb-SanDisk_Cruzer_Blade_00000000000000000000-0:0 \
  --config ./agent.local.toml \
  --yes
```

The command is intentionally destructive. It refuses to run without `--yes`, verifies the destination is a disk, rejects non-removable media unless explicitly allowed, checks optional serial/model/size guardrails, writes the image, injects first-boot assets, runs read-only filesystem checks, and unmounts the drive.

If the image write and byte-compare already succeeded but customization needs to be resumed, skip the slow rewrite:

```bash
./bin/create-agent-boot-drive \
  --destination /dev/disk/by-id/usb-SanDisk_Cruzer_Blade_00000000000000000000-0:0 \
  --config ./agent.local.toml \
  --customize-existing \
  --yes
```

## First Boot

After the USB is written:

1. Shut down the Pi.
2. Boot from the generated USB.
3. Wait for first boot and setup to finish.
4. SSH in from the same LAN:

```bash
ssh my-user@my-agent.local
```

5. Inspect setup health:

```bash
cat ~/CODEX_TODO.md
sudo cat /var/log/codex-agent-health-summary.log
sudo less /var/log/codex-agent-setup.log
```

6. Complete the unavoidable manual Codex auth:

```bash
codex login --device-auth
```

7. Start a fresh Codex session after login so the prewritten YOLO config is active:

```bash
codex-attach
```

## Post-Setup Assertions

The generated image runs pre-auth assertions automatically. See [docs/post-setup-assertions.md](docs/post-setup-assertions.md).

## Node.js Runtime

First boot resolves Node.js from the official distribution index at `https://nodejs.org/dist/index.json` and selects the latest release with an LTS marker. Current releases are ignored until Node.js promotes them to LTS. As of 2026-07-04, that means Node 24 LTS is selected instead of Node 26 Current.

The generated first-boot service clones `Codex-Agent-Setup` and runs its `agent-setup.sh` entrypoint. That setup installs the verified Node.js binary tarball under `/opt/node-lts`, then symlinks `node`, `npm`, `npx`, and `corepack` into `/usr/local/bin`. Debian stable's `nodejs` package can lag behind modern Vite, Vitest, and Supabase tooling, so the boot-drive setup does not use it as the app-work runtime.

Leave `first_boot.node_lts_line` empty to maintain the latest LTS line on rerun. Set it to a major version such as `24` only when deliberately pinning, and remove the pin to resume latest-LTS tracking.
