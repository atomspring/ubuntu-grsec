Ubuntu kernel with grsecurity
=============================

This guide outlines the steps required to compile a kernel for [Ubuntu
Server 14.04 LTS (Trusty Tahr)](http://releases.ubuntu.com/14.04/)
with [Grsecurity](https://grsecurity.net/), for use on servers or desktops.

## Before you begin

The current version of this document assumes you are compiling Linux
kernel version *3.17.2* and Grsecurity version
*grsecurity-3.0-3.17.2-201411091054.patch*. When running commands that include filenames
and/or version numbers, make sure it all matches what you have on your
system.

## Update packages on system

Run the following set of commands to ensure all packages are up to date
on the online and offine servers.


```
sudo apt-get update>/dev/null;sudo apt-get upgrade -y;sudo apt-get dist-upgrade -y
```

## Install dependencies

Run the following command to install the package dependencies required
to compile the new kernel with Grsecurity.

```
sudo apt-get install libncurses5-dev build-essential kernel-package git-core gcc-4.8 gcc-4.8-plugin-dev make attr paxctl
```

Create a directory for Grsecurity and download the public keys that you
will later use to verify the Grsecurity and Linux kernel downloads.

```
mkdir grsec
cd grsec/
wget https://grsecurity.net/spender-gpg-key.asc
gpg --import spender-gpg-key.asc
gpg --recv-key 6092693E
```
## Download kernel and Grsecurity on server

Create a directory for Grsecurity, the Linux kernel, and the other tools
you will be downloading.

```
cd /usr/src
mkdir grsec
cd grsec/
```

Make a copy of the *kernel-package* directory on the online server.

```
cp -a /usr/share/kernel-package ubuntu-package
```

When downloading the Linux kernel and Grsecurity, make sure you get the
long term stable versions and that the version numbers match up.

```
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.17.2.tar.xz
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.17.2.tar.sign
wget https://grsecurity.net/test/grsecurity-3.0-3.17.2-201411091054.patch
wget https://grsecurity.net/test/grsecurity-3.0-3.17.2-201411091054.patch.sig
```

Download the Ubuntu kernel overlay.

```
git clone git://kernel.ubuntu.com/ubuntu/ubuntu-trusty.git
```
### Gather the required files for the Ubuntu kernel overlay

Copy the required directories from the Ubuntu kernel overlay directory
to the correct *ubuntu-package* directory.

```
cp ubuntu-trusty/debian/control-scripts/p* ubuntu-package/pkg/image/
cp ubuntu-trusty/debian/control-scripts/headers-postinst ubuntu-package/pkg/headers/
```

### Verify the digital signatures

Verify the digital signature for Grsecurity.

```
 gpg --verify grsecurity-3.0-3.17.2-201411091054.patch.sig 
```

Verify the digital signature for the Linux kernel.

```
unxz linux-3.2.61.tar.xz
gpg --verify linux-3.17.2.tar.sign
```

Do not move on to the next step until you have successfully verified both
signatures. If either of the signatures fail to verify, re-download both the package and signature and try again.

### Apply Grsecurity patch to the Linux kernel

Extract the Linux kernel archive and apply the Grsecurity patch.

```
tar -xf linux-3.17.2.tar
cd linux-3.17.2/
patch -p1 < ../grsecurity-3.0-3.17.2-201411091054.patch
```

### Configure Grsecurity

Configure Grsecurity with the following command.

```
make menuconfig
```

You will want to follow the steps below to select and configure the
correct options.

 * Navigate to *Security options*
   * Navigate to *Grsecurity*
     * Press *Y* to include it
     * Set *Configuration Method* to *Automatic*
     * Set *Usage Type* to *Server* or *Desktop*
     * Set *Virtualization Type* to *KVM* (if you virtualize)
     * Set *Required Priorities* to *Security* or *Performance*
     * Select *Exit*
   * Select *Exit* 
 * Select *Exit*
 * Select *Yes* to save

You will likely want to boost performance on your kernel a little, besides discluding some packages you don't need (like support for AMD processors if you're on an Intel board).
```
make menuconfig
```
* Navigate to *Processor type and features*
  * Processor Family
   * Select your CPU
  * Deselect stuff like "goldfish", AMD etc
  * For better response on a desktop, select *Preemption model*
   * Select *Preemptible kernel* (Servers should stick with *No forced preemption*)
* Save it and get out of there 

### Compile the kernel with Grsecurity

Use all available cores when compiling the kernel, optimize compilation for local chipset (or any very similar).
```
export CONCURRENCY_LEVEL="$(grep -c '^processor' /proc/cpuinfo)"
export KCFLAGS="-march=native -O2 -pipe" KCPPFLAGS="-march=native -O2 -pipe"
```


Compile the kernel with the Ubuntu overlay. Note that this step may fail
if you are using a small VPS/virtual machine.

```
make-kpkg clean
```
If you want to create a tiny kernel with support for only the hardware you currently have plugged in, run
```
time make localmodconfig
```
Finally, make the kernel and its headers:
```
sudo time make-kpkg --initrd --overlay-dir=../ubuntu-package kernel_image kernel_headers 
```

When the build process is done, you will have the following Debian
packages in the *grsec* directory:

```
linux-headers-3.17.2-grsec_3.17.2-grsec-10.00.Custom_amd64.deb
linux-image-3.17.2-grsec_3.17.2-grsec-10.00.Custom_amd64.deb
```

## !!Sugested!! Set up PaX on Desktops(!) and servers

PaX is the memory-attack blocker, included in Grsecurity. At this point as a desktop user, you'll be setting relaxations of security on many programs which use methods that attackers also use, which gets to be a real pain in the butt. I'd suggest you look into the linux-pax-flags directory on this github for a system which automatically updates PaX flags on a lot of commonly used desktop software.

For a desktop, I'd suggest running the following (note that this app will run automatically after any APT operation, look in /etc/apt/apt.conf.d/99-grsec)
```
git clone https:///github/atomspring/linux-pax-flags
cd linux-pax-flags
./install.sh
linux-pax-flags.sh
```

If you don't want to automatically keep PaX flags updated, run the basics (e.g. for a server)
```
sudo apt-get install paxctl
sudo paxctl -Cpm /usr/sbin/grub-probe
sudo paxctl -Cpm /usr/sbin/grub-mkdevicemap  
sudo paxctl -Cpm /usr/sbin/grub-setup  
sudo paxctl -Cpm /usr/bin/grub-script-check  
sudo paxctl -Cpm /usr/bin/grub-mount  
```

### Install the new kernel with Grsecurity on both servers.

```
sudo dpkg -i *.deb
sudo update-grub
```

### Configure the system to use the grsec kernel by default

Start by finding the exact menuentry name for the kernel.

```
grep menuentry /boot/grub/grub
```

Copy the output and use it in the *sed* command below to set this kernel as the default.

```
sudo sed -i "s/^GRUB_DEFAULT=.*$/GRUB_DEFAULT=2>Ubuntu, with Linux 3.17.2-grsec/" /etc/default/grub
sudo update-grub
sudo reboot
```

### Lock it down

Once you have confirmed that everything works, configure the Grsecurity
lock in *sysctl.conf*.

```
sudo echo "kernel.grsecurity.grsec_lock = 1" >> /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf
```
