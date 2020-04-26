# Orange Livebox to Ubiquiti USG

The goal is to replace the Orange Livebox Play Fibre by a Unify Security Gateway from Ubiquiti.

The services that must work after the replacement are:
- Internet access (IPv4)
- TV (live and services)

However:
- The IPv6 connectivity will not be configured
- The phone service will not be configured

## Hardware

- [Livebox v3 (Play Fibre)](https://assistance.orange.fr/equipement/livebox-et-modems/livebox-play-sagemcom)
- [UniFi Security Gateway](https://eu.store.ui.com/collections/routing-switching/products/unifi-security-gateway)

## Documentation

- [Reference guide [FR|lafibre.info]](https://lafibre.info/remplacer-livebox/le-guide-complet-pour-usgusg-pro-internet-tv-livebox-ipv6/)

## dhclient3

## Background

DHCP packets with a CoS 6 are required in order to obtain an IP address from the Orange DHCP server.
If CoS bits in the the 802.1Q (VLAN) header are not set correctly, the DHCP server of Orange will not answer at all.
This cannot be set updated on the fly by iptables mangle-output because DHCP clients use raw sockets, bypassing the ipfilter chain.
Another option would be to create a rule to tag all trafic with CoS 6, then apply a second rule to tag back all trafic except DHCP with CoS 0, however doing that disables the hardware acceleration.

The proposed solution by [@zoc](https://lafibre.info/profile/zoc/) is to directly patch the *dhclient* sources to inject this CoS 6 priority in the generated DHCP packets.

The patch is availble under *dhclient/iscdhcp_priority.patch*.

## Sources

The *dhclient3* used in the USG is a Vyatta custom version of the Debian Lenny package *dhcp3-3.0.6.dfsg.deb*. [github](https://github.com/vyos-legacy/vyatta-dhcp3)

In order to download the sources used to build the *dhclient3* packaged in the USG firmware, we need to download its corresponding GPL archive from the *Ubiquti* website.

- These firmwares can be found under the [Edge Router Lite section](https://www.ui.com/download/edgemax/edgerouter-lite/erlite3EdgeRouter)

- Select the last available one: **EdgeRouter ERLite-3/ERPoe-5 Firmware v1.10.11**

- Download its [GPL archive](https://dl.ui.com/firmwares/edgemax/v1.10.11/gpl/GPL.ER-e100.v1.10.11.5274249.tar.bz2)

```
wget https://dl.ui.com/firmwares/edgemax/v1.10.11/gpl/GPL.ER-e100.v1.10.11.5274249.tar.bz2
```

- Extract the *vyatta-dhcp3* tarball from the downloaded archive:

```
tar -xzvf GPL.ER-e100.v1.10.11.5274249.tar.bz2 *vyatta-dhcp3*
```

## Build

- The USG CPU is using a MIPS64 architecture [source](https://dl.ubnt.com/datasheets/unifi/UniFi_Security_Gateway_DS.pdf)
- The Operating System seems to be based on Debian 7 (Wheezy) [source](https://help.ui.com/hc/en-us/articles/205202560-EdgeRouter-Add-Debian-Packages-to-EdgeOS)


### QEMU

Using the available [QEMU Debian Wheezy MIPS image](https://people.debian.org/~aurel32/qemu/mips/):
```
$ mkdir mipsbe
$ wget https://people.debian.org/~aurel32/qemu/mips/debian_wheezy_mips_standard.qcow2 -O mipsbe/debian_wheezy_mips_standard.qcow2
$ wget https://people.debian.org/~aurel32/qemu/mips/vmlinux-3.2.0-4-4kc-malta -O mipsbe/vmlinux-3.2.0-4-4kc-malta

$ qemu-system-mips \
  -m 256 \
  -M malta \
  -kernel mipsbe/vmlinux-3.2.0-4-4kc-malta \
  -hda mipsbe/debian_wheezy_mips_standard.qcow2 \
  -append "root=/dev/sda1 console=tty0" \
  -net nic \
  -net user,hostfwd=tcp::2222-:22

# root password
$ ssh root@localhost -p2222
```

### Docker

```
$ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
$ docker run --rm mips64le/debian uname -m
mips64

# should work out of the box but crashes
$ docker run -it --rm multiarch/debian-debootstrap:mips-wheezy /bin/uname -a
# this however works
$ docker run -it --rm multiarch/debian-debootstrap:mips-wheezy /usr/bin/qemu-mips-static /bin/uname -a
mips
# https://github.com/multiarch/qemu-user-static/issues/15
# seems like the issue is that the mips/mipsel entry files are overwritten by the mipsn32/mipsn32el ones

Looking at [qemu-binfmt-conf.sh](https://raw.githubusercontent.com/qemu/qemu/master/scripts/qemu-binfmt-conf.sh), seems like the mipsn32* magic values are the same than mips* ones.

Can we remove the registered qemu-mipsn32/qemu-mipsn32el from /proc/sys/fs/binfmt_misc ?
According to the [developer's guide](https://github.com/multiarch/qemu-user-static/blob/master/docs/developers_guide.md#qemu-user-static-binfmt_misc-and-container) yes, since these files are shared between the host and the containers.

# access the Linux VM storing/running the containers on Mac:
$ screen ~/Library/Containers/com.docker.docker/Data/vms/0/tty
# press enter
# remove the binfmt_misc mipsn32 entry files
$ find /proc/sys/fs/binfmt_misc -type f -name "*mipsn32*" -exec sh -c 'echo -1 > {}' \;
# exit
$ Ctrl-A k
$ y
```