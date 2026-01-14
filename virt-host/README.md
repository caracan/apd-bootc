# Base bootc image

## Build

podman build --pull=newer --cap-add=all --security-opt=label=type:container_runtime_t --device /dev/fuse -f Containerfile -t quay.io/caracan/apd-virt-host:latest .

## Tag

podman tag quay.io/caracan/apd-virt-host:latest quay.io/caracan/apd-virt-host:$(cat version)

## Push

podman push quay.io/caracan/apd-virt-host:latest;
podman push quay.io/caracan/apd-virt-host:$(cat version)

## Build qcow/iso

This currently needs to be done with root

sudo podman pull quay.io/caracan/apd-virt-host:latest
sudo podman run --rm -it --privileged --pull=newer --security-opt label=type:unconfined_t -v ./config.toml:/config.toml:ro -v $(pwd)/output:/output -v /var/lib/containers/storage:/var/lib/containers/storage quay.io/centos-bootc/bootc-image-builder:latest --chown 1000:1000 --type anaconda-iso --rootfs xfs quay.io/caracan/apd-virt-host:latest

sudo podman run --rm -it --privileged --pull=newer --security-opt label=type:unconfined_t -v ./config.toml:/config.toml:ro -v $(pwd)/output:/output -v /var/lib/containers/storage:/var/lib/containers/storage quay.io/centos-bootc/bootc-image-builder:latest --chown 1000:1000 --type qcow2 --rootfs xfs quay.io/caracan/apd-virt-host:latest
