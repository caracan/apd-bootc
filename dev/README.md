# Development Tools bootc Image (`apd-dev`)

Built on top of `apd-virt-host:1`, this image equips a machine with essential command-line developer tooling, infrastructure automation tools, container build utilities, and mesh networking via Tailscale.

## Contents

* **Base Image**: `quay.io/caracan/apd-virt-host:1`
* **Additional Repositories**: Tailscale stable Fedora repository (`tailscale.repo`)
* **Installed Packages**:
  * Version Control: `git`
  * Infrastructure Automation: `ansible`, `ansible-lint`
  * Container & Registry Utilities: `buildah`, `skopeo`, `podman-compose`
  * Networking & Mesh VPN: `tailscale`, `nmap`
  * Terminal Utilities: `tmux`
* **Configuration**:
  * Validated with `bootc container lint`

> **Tip:** Language runtimes and project-specific dependencies are recommended to be run inside [Toolbx](https://containertoolbx.org/) / Podman containers rather than baked directly into the immutable OS image.

## Local Build & Push

### Build
```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
MAJOR=$(jq -r '.major' version.json)

podman build --pull=newer \
  -t quay.io/caracan/apd-dev:latest \
  -t quay.io/caracan/apd-dev:${MAJOR} \
  -t quay.io/caracan/apd-dev:${VERSION} \
  .
```

### Push
```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
MAJOR=$(jq -r '.major' version.json)

podman push quay.io/caracan/apd-dev:latest
podman push quay.io/caracan/apd-dev:${MAJOR}
podman push quay.io/caracan/apd-dev:${VERSION}
```

## Generating Installable Disk Images

Using [`bootc-image-builder`](https://github.com/osbuild/bootc-image-builder):

```bash
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
  quay.io/caracan/apd-dev:1

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
  quay.io/caracan/apd-dev:1
```

## Deployment

On an existing bootc host:
```bash
sudo bootc switch quay.io/caracan/apd-dev:1
```
