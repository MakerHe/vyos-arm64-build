# VyOS ARM64 Build Instructions

## Project Overview

This is a **patch-based build system** for creating VyOS ARM64 images from the official VyOS source. It's NOT a fork - it applies architecture-specific patches to upstream VyOS repositories during the build process.

**Key Architecture:**
- Weekly scheduled builds via GitHub Actions (Fridays @ 01:00 UTC)
- Patches stored in [`data/`](data/) directory, applied to `vyos-build` and `vyos-1x` during build
- Uses custom ARM64 builder container: `ghcr.io/huihuimoe/vyos-arm64-build/vyos-builder:current-arm64`
- Build output: ISO images signed with minisign, published as GitHub releases

## Critical Workflows

### Build Process ([`.github/workflows/auto-build.yml`](.github/workflows/auto-build.yml))

The build happens in three stages within a single job:

1. **Patch Application** (lines 150-200): Clones upstream repos, applies patches from `data/` directory
2. **Package Building** (lines 203-230): Builds specific packages (radvd, vyos-1x, linux-kernel)
3. **ISO Generation** (lines 303-375): Creates generic ARM64 image with custom packages

**Important**: Patches are applied with `patch --no-backup-if-mismatch -p1 -d <target> < data/<patch-file>.patch`

### Builder Image Update ([`.github/workflows/auto-build-builder.yaml`](.github/workflows/auto-build-builder.yaml))

Monthly rebuild (1st @ 01:00 UTC) of the vyos-builder container from official vyos/vyos-build Docker config.

## Patch Management

### Naming Convention
- `vyos-build-NNN-description.patch` - Modifies build system
- `vyos-1x-NNN-description.patch` - Modifies VyOS core logic
- Numbers (001-009) indicate application order; 999 = test/experimental

### Key Patches

**Architecture Fixes:**
- [`001-system_console.patch`](data/vyos-1x-001-system_console.patch): Adds ARM64 serial console support (ttyAMA, ttyFIQ devices)
- [`003-fix_hardcoded_x86_64.patch`](data/vyos-build-003-fix_hardcoded_x86_64.patch): Changes `/usr/lib/x86_64-linux-gnu` → `/usr/lib/aarch64-linux-gnu`

**Boot/Console:**
- [`002-boot_console.patch`](data/vyos-1x-002-boot_console.patch): GRUB template modifications for ARM64 serial console
- [`009-live_boot_serial_console_device.patch`](data/vyos-build-009-live_boot_serial_console_device.patch): Sets serial console device for live boot

**Security/Signing:**
- [`007-no_sbsign.patch`](data/vyos-build-007-no_sbsign.patch): Signs kernel modules but skips vmlinuz signing (ARM64 limitation)
- MOK (Machine Owner Key) support via `data/mok/MOK.key` (secret-based, optional)

### Creating New Patches

1. Clone the upstream repo (vyos-build or vyos-1x)
2. Make changes in a branch
3. Generate patch: `git diff > data/vyos-{build,1x}-NNN-description.patch`
4. Test by running workflow manually with `workflow_dispatch`

## Configuration Files

**Default Configs Embedded in ISO:**
- [`config.boot.default`](data/config.boot.default): Standard config with update-check pointing to this repo
- [`config.boot.dhcp`](data/config.boot.dhcp): Emergency DHCP config for eth0, SSH enabled, credentials: `vyos`/`a_strong-p@ssword`

Users boot with DHCP config via kernel parameter: `vyos-config=/opt/vyatta/etc/config.boot.dhcp`

## Version Management

[`version.json`](version.json): Auto-updated by publish job with latest successful build metadata:
```json
[{
  "url": "https://github.com/.../releases/download/.../vyos-*.iso",
  "version": "YYYY.MM.DD-HHMM-rolling",
  "timestamp": "2026-01-02T02:58:33Z"
}]
```

Referenced by embedded config for update-check functionality.

## Custom Packages

The build includes additional packages beyond standard VyOS (see lines 346-361):
- Tools: `vim-tiny`, `vnstat`, `neofetch`, `tree`, `btop`, `ripgrep`, `ncdu`
- Services: `fastnetmon`, `qemu-guest-agent`
- Security: `mokutil`, `sbsigntool`, `grub-efi-arm64-signed`, `shim-signed`

vnstat configured to use `/opt/vyatta/etc/config/user-data/vnstat` for persistence.

## Dependencies & Mirrors

Environment variables in workflow (lines 19-27):
- `DEBIAN_MIRROR`: Standard Debian repo
- `VYOS_MIRROR`: VyOS package repository (currently `packages.vyos.net/repositories/current/`)

**Note**: Some comments reference old mirror issues - always check if fallback mirrors are still needed.

## Ignored Packages

Lines 247-277 list packages excluded from final ISO:
- Test tools, development packages (-dev, -dbg, -doc suffixes)
- Cloud-specific agents (amazon-cloudwatch, waagent)
- Duplicate/conflicting packages (python3-nftables)

## Troubleshooting

**Build Failures:**
1. Check if upstream changed - patches may need rebasing
2. Review package-build step for dependency issues
3. Verify builder image is current (monthly rebuild)

**Patch Application Fails:**
- Upstream file structure changed - regenerate patch from current upstream
- Check line endings (use `--no-backup-if-mismatch` flag)

**ISO Won't Boot:**
- Serial console device mismatch - verify ttyAMA/ttyFIQ detection in patch 001
- Missing GRUB config - check patch 002 application

## Testing

No automated testing (ARM64 test suite doesn't exist upstream). Manual testing required:
- QEMU with virtio-gpu VGA
- Cloud platforms: Proxmox, HetznerCloud, Scaleway
