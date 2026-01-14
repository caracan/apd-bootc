

podman build --cap-add=all --security-opt=label=type:container_runtime_t --device /dev/fuse -t quay.io/caracan/apd-base .
podman push quay.io/caracan/apd-base:latest 
sudo podman pull quay.io/caracan/apd-base:latest

sudo podman run     --rm     -it     --privileged     --pull=newer     --security-opt label=type:unconfined_t -v ./config.json:/config.json:ro    -v $(pwd)/output:/output     -v /var/lib/containers/storage:/var/lib/containers/storage     quay.io/centos-bootc/bootc-image-builder:latest     --local     --type qcow2 --rootfs xfs    quay.io/caracan/apd-base:latest

sudo podman run     --rm     -it     --privileged     --pull=newer     --security-opt label=type:unconfined_t -v ./config.json:/config.json:ro    -v $(pwd)/output:/output     -v /var/lib/containers/storage:/var/lib/containers/storage     quay.io/centos-bootc/bootc-image-builder:latest     --local     --type anaconda-iso --rootfs xfs    quay.io/caracan/apd-base:latest

sudo virt-install -d    --name fedora-bootc     --cpu host     --vcpus 4     --memory 4096     --import --disk /var/lib/libvirt/images/disk.qcow2,format=qcow2     --os-variant fedora-eln