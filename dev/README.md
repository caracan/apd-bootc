# Development Tools image

This is an image, based on the virt-host image that adds in a set of tools that I use as part of development. Other tools, such as specific language compilers/runtimes, are more likely best worked with the toolbox paradigm.

## Build

```bash
podman build --pull=newer -f Containerfile -t quay.io/caracan/apd-dev:latest .
```

## Tag

```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
podman tag quay.io/caracan/apd-dev:latest quay.io/caracan/apd-dev:${VERSION}
podman tag quay.io/caracan/apd-dev:latest quay.io/caracan/apd-dev:1
```

## Push

```bash
VERSION=$(jq -r '.major + "." + .minor + "." + .zstream' version.json)
podman push quay.io/caracan/apd-dev:latest
podman push quay.io/caracan/apd-dev:${VERSION}
podman push quay.io/caracan/apd-dev:1
```

## Build qcow/iso

This currently needs to be done with root. Create a `config.toml` file from the example or using the [docs](https://bootc-dev.github.io/bootc/bootc-install.html).

Pull the image as root:

```bash
sudo podman pull quay.io/caracan/apd-dev:latest
```

Run one of the following commands depending on the output required:

```bash
sudo podman run --rm -it --privileged --pull=newer --security-opt label=type:unconfined_t -v ./config.toml:/config.toml:ro -v $(pwd)/output:/output -v /var/lib/containers/storage:/var/lib/containers/storage quay.io/centos-bootc/bootc-image-builder:latest --chown 1000:1000 --type anaconda-iso --type qcow2 --rootfs xfs quay.io/caracan/apd-dev:latest
```

```bash
sudo podman run --rm -it --privileged --pull=newer --security-opt label=type:unconfined_t -v ./config.toml:/config.toml:ro -v $(pwd)/output:/output -v /var/lib/containers/storage:/var/lib/containers/storage quay.io/centos-bootc/bootc-image-builder:latest --chown 1000:1000 --type qcow2 --rootfs xfs quay.io/caracan/apd-dev:latest
```
