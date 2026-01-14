
# Bootc Images

This repe contains a number of [bootc](|https://docs.fedoraproject.org/en-US/bootc/) images that provide various functions that I use. Currently they are based on Fedora bootc images.

## Images

### Base

A base image containg the various tools that I would install on all machines.

### Virt Host

Built from the base but adds Virtualisation tools to build a host to run VMs.

### Virt Guest

Built from the base but with the qemu guest tools installed.

### Dev

Built from the virt-host adding in various tools that I use on a machine for development. Other tools are more likely best worked with the toolbox paradigm.

## Building

All builds follow a similar pattern following what is laid out in the [Fedora bootc docs](|https://docs.fedoraproject.org/en-US/bootc/). They are assumed to be ran rootless as much as possible and as bootc improves these they will be updated. Each image has it's own README giving commands to build locally.

In general there are 4 parts

* Build the container
* Tag container
* Push to registry
* Build an installable artifact (Usually anaconda-iso and qemu examples)

The installable artifact is only needed when installing new machines. Bootc also provides other ways of doing this which are described in the [docs](https://bootc-dev.github.io/bootc/bootc-install.html)
