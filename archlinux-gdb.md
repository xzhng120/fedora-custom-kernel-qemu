# ARCH


## on guest
follow https://wiki.archlinux.org/title/Kernel/Traditional_compilation

* `cd <kernel source tree>`
* copy out `/proc/config.gz`  (zcat to kernel/.config)
* `make nconfig` (or menuconfig) and set the kernel local version with something like "-gdb". also make other necessary changes
* make sure to use `modprobed-db` to reduce unnecessary compilation
  * `make LSMOD=\<your modprobed.db\> localmodconfig`

only the following steps are needed for new kernel version
* `make` or `make -j<number of threads>`
* `cp arch/x86/boot/bzImage /boot/vmlinuz_linux_gdb` (suggest not to use dash but instead underscore so grub doens't automatically pick up your kernel)
  * (optional if you want to use direct kernel boot, grub does not support vmlinux, wtf???)
* `make modules_install`
* `mkinitcpio -k 6.4.9-gdb -g /boot/initramfs_linux_gdb.img` (replace major.minor.revision accordingly)
* * (use the -k option matching your new kernel's `uname -r` you can find this based on /lib/modules/6.4.9-gdb)
* `rm -rf /lib/modules/<old.kernel.ver>-gdb` as needed

==========================================================

* open /boot/grub/grub.cfg and find your default boot menuentry, copy to `/etc/grub.d/40_custom`, then modify the `linux` and `initrd` accordingly. When you are done, `grub-mkconfig -o /boot/grub/grub.cfg`
  * doesn't seem necessary, grub seemingly auto detects /boot/vmlinuz-\* (but without nokaslr)
  * optionally add `GRUB_TERMINAL_OUTPUT="console vga_text"` in /etc/default/grub so that you can boot into either kernel on a tty and gpu
  * optional if direct kernel boot
* reboot and select your own kernel on grub

## on host
copy out your kernel tree and do direct kernel boot:
* `rsync -a --info=progress2 root@192.168.122.X:/root/linux <some place on the host>`
* `ln -sf scripts/gdb/vmlinux-gdb.py ./` to improve gdb experience

Sample cmdline, make sure to include nokaslr if you want gdb to work (this is for libvirt xml)
```xml
  <os>
    <type arch='x86_64' machine='pc-q35-rhel9.2.0'>hvm</type>
    <kernel>/path/to/vmlinux</kernel>
    <initrd>/path/to/initramfs-linux-gdb.img</initrd>
    <cmdline>root=UUID=_your_root_uuid rw  loglevel=3 console=ttyS0,115200 nokaslr</cmdline>
  </os>
```

### avoid compiling on guest and then rsync'ing to host the compiled kernel tree

You should be able to just simply run `make` on host to compile the kernel with modules

Assuming you are unpriviledged on host in the kernel tree root and have the initramfs from the official build or your own in [on guest](#on-guest)

follow these steps to replace the kernel modules with your host built ones.

* `unshare -r` to fake as root
* `file initramfs_linux_gdb.img` -> initramfs_linux_gdb.img: Zstandard compressed data (v0.8+), Dictionary ID: None
* ` zstdcat initramfs_linux_gdb.img > initramfs_decomp`
* `file initramfs_decomp` -> initramfs_decomp: ASCII cpio archive (SVR4 with no CRC)
* `mkdir initramfs_root`
  * `cd initramfs_root`
  * `cpio -i < ../initramfs_decomp`
  * `cd ..` 
* `cd initramfs_root/usr/lib/modules/<uname -r of new kernel>/kernel`
  * `find -name *.ko -exec cp ../../../../../../{} {} \;` to replace all of the *.ko
  * `cd ../../../../../` (back to the initramfs_root directory)
  * `find . | cpio -o -c > ../initramfs_decomp`
  * `file ../initramfs_decomp` verify that it spits out the same information
  * `cd ..` 
* `zstd initramfs_decomp -o initramfs_linux_gdb.img`
  * `file initramfs_linux_gdb.img` also verify
* `exit` from userns

***Do Direct Kernel Boot!!!***

you may have to redo [on guest](#on-guest) after many updates
