# Base bootc Image (`apd-base`)

The foundational bootc container image from which all other images in this repository derive. It installs the essential system packages, services, and administrative defaults used across all machines.

## Contents

* **Base OS**: `quay.io/fedora/fedora-bootc:43` (pinned upstream digest tracked by Renovate)
* **Installed Packages**:
  * Networking & Remote Access: `NetworkManager`, `openssh-server`, `wget`, `nfs-utils`
  * Container Tools: `podman`, `systemd-container`
  * Web Administration: `cockpit`, `cockpit-podman`
  * System & Shell Utilities: `sudo`, `firewalld`, `acl`, `bash-color-prompt`
* **Configuration & Services**:
  * `cockpit.socket` enabled at boot
  * Firewalld pre-configured to allow Cockpit (`firewall-offline-cmd --add-service cockpit`)
  * Passwordless sudo enabled for `%wheel` via `/etc/sudoers.d/wheel-nopasswd` (mode `0440`)

## Local Build & Push

### Build

```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
MAJOR=$(jq -r '.major' version.json)

podman build --pull=newer \
  -t quay.io/caracan/apd-base:latest \
  -t quay.io/caracan/apd-base:${MAJOR} \
  -t quay.io/caracan/apd-base:${VERSION} \
  .
```

### Push

```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
MAJOR=$(jq -r '.major' version.json)

podman push quay.io/caracan/apd-base:latest
podman push quay.io/caracan/apd-base:${MAJOR}
podman push quay.io/caracan/apd-base:${VERSION}
```

## Generating Installable Disk Images

Using [`bootc-image-builder`](https://github.com/osbuild/bootc-image-builder):

```bash
# Prepare config.toml
cp config.toml.example config.toml
# Edit config.toml to add your SSH key

# Create QCOW2 virtual machine disk
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

# Create Anaconda Installation ISO (bare-metal)
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

## Deployment

On an existing bootc host:

```bash
sudo bootc switch quay.io/caracan/apd-base:1
```
