# Virtualization Guest bootc Image (`apd-virt-guest`)

Built on top of `apd-base:1`, this image is tailored for running as a lightweight virtual machine guest inside KVM/QEMU, Proxmox, or other hypervisors.

## Contents

* **Base Image**: `quay.io/caracan/apd-base:1`
* **Installed Packages**:
  * Guest Agent: `qemu-guest-agent`
  * Container Orchestration: `podman-compose`
* **Configuration & Services**:
  * `qemu-guest-agent` service enabled at boot (enables host-to-guest shutdown, IP reporting, and fsfreeze for live backups)

## Local Build & Push

### Build

```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
MAJOR=$(jq -r '.major' version.json)

podman build --pull=newer \
  -t quay.io/caracan/apd-virt-guest:latest \
  -t quay.io/caracan/apd-virt-guest:${MAJOR} \
  -t quay.io/caracan/apd-virt-guest:${VERSION} \
  .
```

### Push

```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
MAJOR=$(jq -r '.major' version.json)

podman push quay.io/caracan/apd-virt-guest:latest
podman push quay.io/caracan/apd-virt-guest:${MAJOR}
podman push quay.io/caracan/apd-virt-guest:${VERSION}
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
  quay.io/caracan/apd-virt-guest:1

# Create Anaconda Installation ISO
sudo podman run --rm -it --privileged --pull=newer \
  --security-opt label=type:unconfined_t \
  -v ./config.toml:/config.toml:ro \
  -v $(pwd)/output:/output \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  --chown 1000:1000 \
  --type anaconda-iso \
  --rootfs xfs \
  quay.io/caracan/apd-virt-guest:1
```

## Spawning a VM with virt-install

Once the `.qcow2` image is created in `output/qcow2/disk.qcow2`:

```bash
sudo virt-install \
  --name apd-guest-01 \
  --cpu host \
  --vcpus 2 \
  --memory 2048 \
  --import \
  --disk output/qcow2/disk.qcow2,format=qcow2 \
  --os-variant fedora-unknown \
  --network default
```

## Deployment

On an existing bootc host:

```bash
sudo bootc switch quay.io/caracan/apd-virt-guest:1
```
