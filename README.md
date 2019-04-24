# How to enable sound for the Asus E200HA ultrabook in the Fedora kernel

For the time being I will only provide the instructions for building the kernel. In the future I will also provide the dockerfile to build the kernel using docker.

These instructions have been verified for Fedora 29 and kernel 5.0.9 (x86_64).

In order to enable sound for this system, I've taken code from https://github.com/heikomat/linux/tree/cx2072x_5.0 and https://github.com/conexant/codec_drivers/tree/rpi-4.4.y. The former provides the sources for the Cherrytrail with CX2072X codec Intel Machine driver for and the latter provides the sources for the Conexant CX2072 codec. 

https://github.com/heikomat/linux/tree/cx2072x_5.0 also provides the sources for the Conexant CX2072 codec but I preferred to use the Conexant source code directly. In order for that to work, I changed ```enable_detect(component)``` to ```enable_jack_detect(component)``` in https://github.com/heikomat/linux/blob/cx2072x_5.0/sound/soc/intel/boards/cht_cx2072x.c.

Let's start by setting up fedora for building the kernel (from https://fedoraproject.org/wiki/Building_a_custom_kernel):
```
sudo dnf install fedpkg fedora-packager rpmdevtools ncurses-devel pesign
```
Add the user doing the build to /etc/pesign/users and run the authorize user script:
```
sudo /usr/libexec/pesign/pesign-authorize-users
```
Checkout the source of the kernel anonymously: 
```
fedpkg clone -a kernel
cd kernel
```
Switch to the Fedora 29 branch (type fedpkg switch-branch to list all the different branches):
```
fedpkg switch-branch f29
```
Move the patch ```0001-set-configs-for-cx2072x.patch``` to the current directory and apply it:
```
git apply 0001-set-configs-for-cx2072x.patch
```
This sets the proper configurations in the configs directory for compiling the kernel with cx2072x codec support. Rebuild the configs for the kernels:
```
./build_configs.sh
```
and update the ```kernel.spec``` to include ```0001-asus-e200ha-Enable-sound.patch``` (do not apply this patch, just move it to the directory):
```
./scripts/newpatch.sh 0001-asus-e200ha-Enable-sound.patch
```
Change the buildid in the ```kernel.spec``` file from ```0001_asus_e200ha_Enable_sound.patch``` to a small identifier (if you do not do this the build will fail because the identifier is too long). I used the identifier ```cht```:
```
sed -i 's/buildid .0001_asus_e200ha_Enable_sound.patch/buildid .cht/g' kernel.spect
```
Fetch any dependencies need for compilation:
```
sudo dnf builddep kernel.spec
```
Optionally, you can configure the kernel at this point to remove drivers and so on:
```
cd /usr/src/kernels/$(uname -r)
zcat /proc/config.gz >.config
make oldconfig
make menuconfig
```
Now you are ready to compile. You can either use ```fast-build.sh```:
```
fedpkg srpm
./scripts/fast-build.sh x86_64 kernel-{the version of the kernel to be compiled and buildid}.src.rpm
```
to build the kernel without debugging information, perf or tools. Otherwise execute:
```
fedpkg local
```
to build both debug and release kernels.

**If you build the kernel with ```fedpkg local``` and you haven't changed the configuration of the kernel to reduce the drivers that are going to be built, you will need at least 64GB of space.** 

In the first case the resulting rpms reside in ```~/rpmbuild/RPMS/x86_64``` and the second case they reside in ```kernel/x86_64```.

Now you can install the kernel with ```dnf install kernel*.rpm```. Be sure that you have the ```linux-firmware``` package installed.

Lastly, update your grub configuration with ```sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg```. Reboot and hopefully, enjoy.

### Last Remarks
You will probably get two warnings when compiling the modules for the cx2072x codec, I will look into them in the future.
