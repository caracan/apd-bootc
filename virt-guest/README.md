# Virtual Guest bootc image

This builds on the apd-base image adding in qemu guest agent. 

## Build

```bash
podman build --pull=newer -f Containerfile -t quay.io/caracan/apd-virt-guest:latest .
```

## Tag

```bash
podman tag quay.io/caracan/apd-virt-guest:latest quay.io/caracan/apd-virt-guest:$(cat version)
```

## Push

```bash
podman push quay.io/caracan/apd-virt-guest:latest;
podman push quay.io/caracan/apd-virt-guest:$(cat version)
```

## Build qcow/iso

This currently needs to be done as root. Create a config.toml file from the example or using the [docs](https://bootc-dev.github.io/bootc/bootc-install.html)

Pull the image as root

```bash
sudo podman pull quay.io/caracan/apd-virt-guest:latest
```

```bash
sudo podman run --rm -it --privileged --pull=newer --security-opt label=type:unconfined_t -v ./config.toml:/config.toml:ro -v $(pwd)/output:/output -v /var/lib/containers/storage:/var/lib/containers/storage quay.io/centos-bootc/bootc-image-builder:latest --chown 1000:1000 --type qcow2 --rootfs xfs quay.io/caracan/apd-virt-guest:latest
```

```bash
sudo podman run --rm -it --privileged --pull=newer --security-opt label=type:unconfined_t -v ./config.toml:/config.toml:ro -v $(pwd)/output:/output -v /var/lib/containers/storage:/var/lib/containers/storage quay.io/centos-bootc/bootc-image-builder:latest --chown 1000:1000 --type anaconda-iso --rootfs xfs quay.io/caracan/apd-virt-guest:latest
```

## Deploy from qcow

```bash
sudo virt-install -d    --name fedora-bootc     --cpu host     --vcpus 4     --memory 4096     --import --disk /var/lib/libvirt/images/disk.qcow2,format=qcow2     --os-variant fedora-unknown
```
