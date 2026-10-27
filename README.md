# Bootc Images

A collection of custom [bootc](https://docs.fedoraproject.org/en-US/bootc/) container images based on Fedora, providing pre-configured operating system trees for bare-metal and virtual machine deployments.

## Image Hierarchy

The images are structured in a layered dependency hierarchy:

```
quay.io/fedora/fedora-bootc:43 (Upstream)
              │
          apd-base
         ╱        ╲
  apd-virt-host  apd-virt-guest
         │
      apd-dev
```

## Image Catalog

| Image | Base Image | Key Packages & Features | Registry URL |
| :--- | :--- | :--- | :--- |
| **`apd-base`** | `fedora-bootc:43` | Cockpit, Cockpit-Podman, Podman, SSH, Sudo (wheel NOPASSWD), Firewalld, NFS utils | [`quay.io/caracan/apd-base`](https://quay.io/repository/caracan/apd-base) |
| **`apd-virt-host`** | `apd-base:1` | KVM/QEMU, Libvirt, Virt-Install, Cockpit-Machines, `libvirt-guests` service | [`quay.io/caracan/apd-virt-host`](https://quay.io/repository/caracan/apd-virt-host) |
| **`apd-virt-guest`** | `apd-base:1` | QEMU Guest Agent (enabled), Podman-Compose | [`quay.io/caracan/apd-virt-guest`](https://quay.io/repository/caracan/apd-virt-guest) |
| **`apd-dev`** | `apd-virt-host:1` | Git, Ansible, Ansible-Lint, Buildah, Skopeo, Tmux, Nmap, Tailscale, Podman-Compose | [`quay.io/caracan/apd-dev`](https://quay.io/repository/caracan/apd-dev) |

## Tagging & Versioning

Each image directory contains a `version.json` specifying semantic versioning:
* **`major`**: Bounded to the major OS release (e.g., `1` for Fedora 43). Downstream images lock to `:1` to prevent unexpected major OS upgrades.
* **`minor`**: Feature additions or package set changes.
* **`zstream`**: Patch updates, rebuilds, and upstream security fixes.

Images are pushed to Quay with the following tags:
* `:latest` — The newest build.
* `:1` — The latest build of major release 1.
* `:1.X` — The latest build of minor release 1.X.
* `:1.X.Y` — Specific immutable version build.

## CI/CD & Automation

1. **Upstream Monitoring (Renovate)**:
   * Renovate monitors upstream `quay.io/fedora/fedora-bootc:43` digest changes.
   * Internal images (`quay.io/caracan/*`) are ignored by Renovate to avoid redundant polling and PR cascades.
2. **PR Verification & Cascading Version Bumps**:
   * When Renovate opens a PR, `.github/workflows/pr-version-check.yaml` runs.
   * It performs a dry-run test build of modified images with `buildah` (and runs `bootc container lint`).
   * It cascades `zstream` bumps across all affected downstream images directly in the PR branch.
   * Auto-merge merges the PR into `main` once checks succeed.
3. **Cascading Build Pipeline**:
   * On merge to `main`, `.github/workflows/build-images.yaml` builds and pushes the updated images in dependency order:
     `base` $\rightarrow$ `virt-host` / `virt-guest` $\rightarrow$ `dev`.
   * Newly pushed `:1` tags are immediately consumed by dependent builds.

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
podman build --pull=newer \
  -t quay.io/caracan/apd-base:latest \
  -t quay.io/caracan/apd-base:1 \
  -t quay.io/caracan/apd-base:${VERSION} .
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
     quay.io/caracan/apd-base:1
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
     quay.io/caracan/apd-base:1
   ```

## Deploying & Updating Running Systems

### Switch to an Image
On any running bootc host, switch to one of these images:
```bash
sudo bootc switch quay.io/caracan/apd-base:1
```

### Upgrade System
To pull the latest updates within the tracked stream:
```bash
sudo bootc upgrade
```

### Check Status
```bash
sudo bootc status
```
