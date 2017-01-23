# LXC 101: Creating and Managing Linux Containers

Linux Containers (LXC) is a userspace interface for the Linux kernel
containment features, providing a software virtualization solution that
relies on many isolated instances (containers) sharing the same Linux
kernel and memory at the same time, having limited access to the network
and strict limits on memory and file system access.

It is commonly used for running microservices isolated from each other
that communicate on the same network, or for providing secure Linux
virtual guests for performing risky activities and confining any issues
within an instance.


## Prerequisites

- Ubuntu version 14.04 or 16.04, LTS 64-bit
- Kernel 3.18 or later
- `sudo` configured to allow you to perform administrative tasks using
  your password
- The more RAM you have, the better


## Installation

Open up your terminal emulator and type:

```sh
$ sudo apt-get update
$ sudo apt-get install lxc lxc-templates
```

Enter your password when requested from you, and LXC will be installed.


# Details about containers

LXC provides `templates`, a set of instructions usually written in Bash
(they can be in any programming language) that download packages and
prepare an installation of a Linux distribution in a directory. They are
used to create a system which will run in a LXC container.

Containers made using Ubuntu or Debian templates use `debootstrap` to
bootstrap an up-to-date system from downloaded packages. Once built, the
installation is cached and kept in `/var/cache/lxc/ubuntu_version_name/
rootfs-amd64`. The configuration data for each container is kept in
`/var/lib/lxc/` in a directory with the same name as the container.
There is the `config` file used to store configuration data, and the
`rootfs/` directory where that container's files are kept.

Default configuration data for all containers used at their creation is
stored in `/etc/lxc/default.conf` or in `/etc/lxc/lxc.conf`.

LXC provides two layers, the base one contains the base directory of the
root file system of a distribution, and the second one provides an
overlay file system which contains the changes made to the base layer
during the execution of an LXC container.


## Usage

- Create a container named `p1` using a template called `ubuntu`:
```sh
$ sudo lxc-create -t ubuntu -n p1
```
- Start the container named `p1` daemonized (in the background):
```sh
$ sudo lxc-start -n p1 -d
```
- Enter the container named `p1` using the console and log in:
  - Detach from the session using *Ctrl+A Q*
```sh
$ sudo lxc-console -n p1
```
- Spawn `bash` directly in the container named `p1`, bypassing login:
```sh
$ sudo lxc-attach -n p1
```
- Get info about the container, including its IP address, so you can
  SSH directly in it as user `ubuntu`:
```sh
$ sudo lxc-info -n p1
$ ssh ubuntu@ip_address_of_p1
```
- Stopping the container named `p1` cleanly:
  - Same as issuing `sudo poweroff` in the container
```sh
$ sudo lxc-stop -n p1
```
- Killing the container named `p1` (immediate, dirty shutdown):
  - Use it only if the container cannot be accessed by any means and is
    unresponsive and using up all resources; important data within it
    may be lost or damaged, take care!
```sh
$ sudo lxc-stop -n p1 -k
```
