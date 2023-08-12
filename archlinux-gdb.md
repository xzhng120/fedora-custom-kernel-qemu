# ARCH


## on guest
follow https://wiki.archlinux.org/title/Kernel/Traditional_compilation

* cd \<kernel source tree\>
* copy out /proc/config.gz  (zcat to kernel/.config)
* make nconfig (or menuconfig) and set the kernel local version with something like "-gdb". also make other necessary changes
* make sure to use modprobed-db to reduce unnecessary compilation
* * make LSMOD=\<your modprobed.db\> localmodconfig

only the following steps are needed for new kernel version
* make
* cp arch/x86/boot/bzImage /boot/vmlinuz_linux_gdb (suggest not to use dash but instead underscore so grub doens't automatically pick up your kernel)
* * (optional if you want to use direct kernel boot, grub does not support vmlinux, wtf???)
* make modules_install
* mkinitcpio -k 6.4.9-gdb -g /boot/initramfs_linux_gdb.img (replace major.minor.revision accordingly)
* * (use the -k option matching your new kernel's $(uname -r) you can find this based on /lib/modules/6.4.9-gdb)
* rm -rf /lib/modules/\<old.kernel.ver\>-gdb

==========================================================

* open /boot/grub/grub.cfg and find your default boot menuentry, copy to /etc/grub.d/40_custom, then modify the `linux` and `initrd` accordingly. When you are done, grub-mkconfig -o /boot/grub/grub.cfg 
* * doesn't seem necessary, grub seemingly auto detects /boot/vmlinuz-\* (but without nokaslr)
* * optionally add GRUB_TERMINAL_OUTPUT="console vga_text" in /etc/default/grub so that you can boot into either kernel on a tty
* * optional if direct kernel boot
* reboot and select your own kernel on grub

## on host
copy out your kernel tree and do direct kernel boot:
* rsync -a --info=progress2 root@192.168.122.X:/root/linux <some place on the host>
* ln -sf scripts/gdb/vmlinux-gdb.py ./

Sample cmdline, make sure to include nokaslr if you want gdb to work (this is for libvirt xml)
```xml
  <os>
    <type arch='x86_64' machine='pc-q35-rhel9.2.0'>hvm</type>
    <kernel>/path/to/vmlinux</kernel>
    <initrd>/path/to/initramfs-linux-gdb.img</initrd>
    <cmdline>root=UUID=_your_root_uuid rw  loglevel=3 console=ttyS0,115200 nokaslr</cmdline>
  </os>
```
