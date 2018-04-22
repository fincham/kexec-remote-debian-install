# kexec-remote-debian-install

Work in progress tools to remotely re-install Debian or Ubuntu using kexec and SSH.

## build.py

Before running `build.py` you will need the `linux` and `initrd.gz` files from the netboot installer of the distro you prefer.

* Ubuntu Bionic AMD64 http://archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/current/images/netboot/ubuntu-installer/amd64/
* Debian Stretch AMD64 http://deb.debian.org/debian/dists/stretch/main/installer-amd64/current/images/netboot/debian-installer/amd64/

If you're upgrading from Debian Jessie to Debian Stretch you may not know what "eth0" is going to become inside the Stretch installer, as Stretch by default uses `systemd`'s new interface naming scheme. In order to predict what "eth0" will become once you reboot, you can examine the output of `udevadm` like so:

    udevadm test /sys/class/net/eth0 2>/dev/null | grep ID_NET_NAME=

Occasionally for reasons as yet unclear the device will end up with its `ID_NET_NAME_PATH` (e.g. `enp0s25` or similar) rather than its `ID_NET_NAME`. If the script doesn't seem to bring up networking during a Jessie -> Stretch re-install then it would be a good idea to try the other possible name format.

Currently `build.py` must be run as root, as it unpacks the cpio archive and re-packs it, and this allows the permissions of various device files to be preserved. 

    $ sudo ./build.py initrd.gz eth0 172.16.0.4 255.255.255.0 172.16.0.1 8.8.8.8

    d-i anna/choose_modules string network-console
    d-i preseed/early_command string anna-install network-console

    d-i network-console/password password c96f4ef5edef6a2daa7b63eb81db5f66
    d-i network-console/password-again password c96f4ef5edef6a2daa7b63eb81db5f66

    d-i netcfg/choose_interface select eth0
    d-i netcfg/disable_dhcp boolean true
    d-i netcfg/get_ipaddress string 172.16.0.4
    d-i netcfg/get_netmask string 255.255.255.0
    d-i netcfg/get_gateway string 172.16.0.1
    d-i netcfg/get_nameservers string 8.8.8.8
    d-i netcfg/confirm_static boolean true

    d-i debian-installer/locale string en_NZ.UTF-8
    d-i console-keymaps-at/keymap select us
        
    Unpacking initrd...
    105824 blocks

    Re-packing initrd...
    105825 blocks

Once `initrd.gz` has been re-packed the new kernel can be launched directly with kexec:

    kexec --command-line="auto=true priority=critical" --initrd=initrd.gz linux

Once the machine has booted the new kernel, the installer will be available over SSH on the IP address given on the command line, with the username `installer` and the password shown during the build output. The password is generated randomly every time the script is run.

## Tested distributions and releases

This script is known to be able to install:

* Debian Jessie
* Debian Stretch
* Ubuntu Trusty
* Ubuntu Xenial
* Ubuntu Bionic

And will probably work on other releases too.

## Caveats

Presently the mirror used after rebooting may default to one that is not optimal for your location, so the initial bringup of SSH will take a worryingly long time. Don't worry, if you can ping it it's probably going to get there in the end.

The method this uses to take over the initrd is really a horrible hack. One day I'll make something more structured.
