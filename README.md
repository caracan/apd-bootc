# Bootc Images

A collection of custom [bootc](https://docs.fedoraproject.org/en-US/bootc/) container images based on Fedora, providing pre-configured operating system trees for bare-metal and virtual machine deployments.

## Image Hierarchy

The images are structured in a layered dependency hierarchy:

```text
quay.io/fedora/fedora-bootc:44 (Upstream)
              │
          apd-base
         ╱        ╲
  apd-virt-host  apd-virt-guest
         │
      apd-dev
```

## Image Catalog

| Image | Base Image | Key Packages & Features | TuneD Profile | Registry URL |
| :--- | :--- | :--- | :--- | :--- |
| **`apd-base`** | `fedora-bootc:44` | Cockpit, Cockpit-Podman, Podman, SSH, Sudo (wheel NOPASSWD), Polkit (machinectl shell), Firewalld, NFS utils, TuneD, auto-updates masked | `throughput-performance` | [`quay.io/caracan/apd-base`](https://quay.io/repository/caracan/apd-base) |
| **`apd-virt-host`** | `apd-base:2` | KVM/QEMU, Libvirt, Virt-Install, Cockpit-Machines, `libvirt-guests` service | `virtual-host` | [`quay.io/caracan/apd-virt-host`](https://quay.io/repository/caracan/apd-virt-host) |
| **`apd-virt-guest`** | `apd-base:2` | QEMU Guest Agent (enabled), Podman-Compose | `virtual-guest` | [`quay.io/caracan/apd-virt-guest`](https://quay.io/repository/caracan/apd-virt-guest) |
| **`apd-dev`** | `apd-virt-host:2` | Git, Ansible, Ansible-Lint, Buildah, Skopeo, Tmux, Nmap, Tailscale, Podman-Compose, Just | `throughput-performance` | [`quay.io/caracan/apd-dev`](https://quay.io/repository/caracan/apd-dev) |

## Tagging & Versioning

Each image directory contains a `version.json` specifying semantic versioning:

* **`major`**: Bounded to the major OS release (e.g., `1` for Fedora 43, `2` for Fedora 44). Downstream images lock to the major tag (e.g., `:2`) to prevent unexpected major OS upgrades.
* **`minor`**: Feature additions or package set changes.
* **`zstream`**: Patch updates, rebuilds, and upstream security fixes.

Images are pushed to Quay with the following tags:

* `:latest` — The newest build.
* `:<major>` (e.g., `:2`) — The latest build of that major release.
* `:<major>.<minor>` (e.g., `:2.1`) — The latest build of that minor release.
* `:<major>.<minor>.<zstream>` (e.g., `:2.1.0`) — Specific immutable version build.

## CI/CD & Automation

1. **Upstream Monitoring (Renovate)**:
   * Renovate monitors upstream `quay.io/fedora/fedora-bootc:44` digest changes.
   * Internal images (`quay.io/caracan/*`) are ignored by Renovate to avoid redundant polling and PR cascades.
2. **PR Verification & Cascading Version Bumps**:
   * When Renovate opens a PR, `.github/workflows/pr-version-check.yaml` runs.
   * It performs a dry-run test build of modified images with `buildah` (and runs `bootc container lint`).
   * It cascades `zstream` bumps across all affected downstream images directly in the PR branch.
   * Auto-merge merges the PR into `main` once checks succeed.
3. **Cascading Build Pipeline**:
   * On merge to `main`, `.github/workflows/build-images.yaml` builds and pushes the updated images in dependency order:
     `base` $\rightarrow$ `virt-host` / `virt-guest` $\rightarrow$ `dev`.
   * Newly pushed `major` tags are immediately consumed by dependent builds.

## Local Development & Building

### Build All Images

To build and tag all images locally in dependency order:

```bash
./build-all
```

### Build a Single Image

To build an individual image manually (for example, `base`):

```bash
cd base
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
MAJOR=$(jq -r '.major' version.json)
MINOR=$(jq -r '.minor' version.json)

podman build --pull=newer \
  -t quay.io/caracan/apd-base:latest \
  -t quay.io/caracan/apd-base:${MAJOR} \
  -t quay.io/caracan/apd-base:${MAJOR}.${MINOR} \
  -t quay.io/caracan/apd-base:${VERSION} \
  .
```

## Creating Installable Artifacts (Disk Images & ISOs)

Installable disk images (QCOW2) or Anaconda installation ISOs can be generated using [`bootc-image-builder`](https://github.com/osbuild/bootc-image-builder).

1. Copy the example config:

   ```bash
   cp base/config.toml.example config.toml
   ```

2. Edit `config.toml` to add your user account and SSH public key:

   ```toml
   [[customizations.user]]
   name = "apd"
   key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5..."
   groups = ["wheel"]
   ```

3. Generate a QCOW2 virtual machine disk:

   ```bash
   sudo podman run --rm -it --privileged --pull=newer \
     --security-opt label=type:unconfined_t \
     -v ./config.toml:/config.toml:ro \
     -v $(pwd)/output:/output \
     -v /var/lib/containers/storage:/var/lib/containers/storage \
     quay.io/centos-bootc/bootc-image-builder:latest \
     --chown 1000:1000 \
     --type qcow2 \
     --rootfs xfs \
     quay.io/caracan/apd-base:2
   ```

4. Generate an Anaconda installation ISO (for bare-metal install):

   ```bash
   sudo podman run --rm -it --privileged --pull=newer \
     --security-opt label=type:unconfined_t \
     -v ./config.toml:/config.toml:ro \
     -v $(pwd)/output:/output \
     -v /var/lib/containers/storage:/var/lib/containers/storage \
     quay.io/centos-bootc/bootc-image-builder:latest \
     --chown 1000:1000 \
     --type anaconda-iso \
     --rootfs xfs \
     quay.io/caracan/apd-base:2
   ```

## System Performance Tuning (TuneD)

All images include [TuneD](https://tuned-project.org/) to automatically tune kernel parameters, sysctls, I/O schedulers, and CPU governors for specific server roles.

Because container builds do not run a live D-Bus daemon, the profile is baked statically during image build into `/etc/tuned/active_profile` and enabled via `systemctl enable tuned`:

* **`throughput-performance`** (`apd-base`, `apd-dev`): Optimizes CPU governors, dirty page writebacks, and I/O readahead for server throughput and compute-heavy compilation/container builds.
* **`virtual-host`** (`apd-virt-host`): Optimizes KVM/QEMU hypervisors (memory overcommit, KVM halt-polling, dirty page thresholds, and scheduler latencies under multi-VM contention).
* **`virtual-guest`** (`apd-virt-guest`): Optimizes VM guests (virtio disk elevators, dirty page flush intervals, avoiding host/guest scheduling conflicts).

To inspect the active profile on a running host:

```bash
tuned-adm active
```

## Deploying & Updating Running Systems

### Automatic Updates Masked by Default

By default in `fedora-bootc`, `bootc-fetch-apply-updates.timer` checks for updates and reboots the system automatically. In this repository, **`bootc-fetch-apply-updates.timer` is masked in `apd-base`** (and inherited by all images) to prevent surprise reboots and disruption of running workloads.

### Update Options for Deployed Machines

Depending on your operational requirements, choose one of the following approaches to manage updates:

#### Option 1: Ad-Hoc / Orchestrated Updates (Ansible, Scripts, Manual)

Manually or via configuration management, pull and apply updates during planned maintenance:

```bash
# Check and download/stage the update (requires manual reboot later):
sudo bootc upgrade

# Check, stage, and reboot immediately to apply:
sudo bootc upgrade --apply

# Check current status:
sudo bootc status
```

#### Option 2: Unmask & Apply a Custom Schedule Drop-in

If you want a machine to automatically update on a predictable schedule (e.g., every Sunday at 3:00 AM), deploy a drop-in override to `/etc/systemd/system/bootc-fetch-apply-updates.timer.d/override.conf`:

```ini
[Unit]
Description=Weekly bootc update schedule

[Timer]
# Clear upstream triggers
OnBootSec=
OnUnitActiveSec=
OnCalendar=

# Define custom maintenance window
OnCalendar=Sun *-*-* 03:00:00
Persistent=true
```

Then unmask and enable the timer:

```bash
sudo systemctl unmask bootc-fetch-apply-updates.timer
sudo systemctl daemon-reload
sudo systemctl enable --now bootc-fetch-apply-updates.timer
```

#### Option 3: Dedicated Standalone Maintenance Timer (Clean Separation)

Leave the upstream `bootc-fetch-apply-updates.timer` masked, and deploy a custom systemd unit pair to `/etc/systemd/system/`:

1. `/etc/systemd/system/bootc-scheduled-upgrade.service`:

   ```ini
   [Unit]
   Description=Scheduled System Update (bootc)
   
   [Service]
   Type=oneshot
   ExecStart=/usr/bin/bootc upgrade --apply
   ```

2. `/etc/systemd/system/bootc-scheduled-upgrade.timer`:

   ```ini
   [Unit]
   Description=Run bootc upgrade on schedule
   
   [Timer]
   OnCalendar=Sun *-*-* 03:00:00
   Persistent=true
   
   [Install]
   WantedBy=timers.target
   ```

Enable and start the custom timer:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bootc-scheduled-upgrade.timer
```

### Switching to an Image

On any running bootc host, switch to one of these images:

```bash
sudo bootc switch quay.io/caracan/apd-base:2
```
