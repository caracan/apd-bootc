# Virtualization Host bootc Image (`apd-virt-host`)

Built on top of `apd-base:1`, this image configures a dedicated virtualization host capable of running and managing KVM virtual machines.

## Contents

* **Base Image**: `quay.io/caracan/apd-base:1`
* **Installed Packages**:
  * Hypervisor & Drivers: `qemu-kvm`, `libvirt`
  * Management Tools: `virt-install`, `virt-viewer`
  * Web Interface: `cockpit-machines`
* **Configuration & Services**:
  * `libvirt-guests` service enabled at boot (graceful suspension/shutdown of guests on host reboot)
  * System `qemu` user configured (`/etc/passwd`)
  * Validated with `bootc container lint`

## Local Build & Push

### Build
```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
MAJOR=$(jq -r '.major' version.json)

podman build --pull=newer \
  -t quay.io/caracan/apd-virt-host:latest \
  -t quay.io/caracan/apd-virt-host:${MAJOR} \
  -t quay.io/caracan/apd-virt-host:${VERSION} \
  .
```

### Push
```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
MAJOR=$(jq -r '.major' version.json)

podman push quay.io/caracan/apd-virt-host:latest
podman push quay.io/caracan/apd-virt-host:${MAJOR}
podman push quay.io/caracan/apd-virt-host:${VERSION}
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
  quay.io/caracan/apd-virt-host:1

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
  quay.io/caracan/apd-virt-host:1
```

## Deployment

On an existing bootc host:
```bash
sudo bootc switch quay.io/caracan/apd-virt-host:1
```
