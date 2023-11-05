# Archlinux QEMU-GDB guide

## prerequisite

An archlinux VM managed by libvirt

## Shared the source between host and guest

create a shared directory on the host for the source then share it with the guest using virtiofs
```xml
...
    <filesystem type='mount' accessmode='passthrough'>
      <driver type='virtiofs'/>
      <source dir='<kernel source tree>'/>
      <target dir='kernel-build'/>
      <address type='pci' domain='...' bus='...' slot='...' function='...'/>
    </filesystem>
...
```
mount on guest by adding to `/etc/fstab`:
`kernel-build            /root/linux-gdb virtiofs rw,relatime`

## Compile the kernel

Although doing all of the following on the guest should also work, however, it makes sense to use the host if not all CPUs are allocated to the guest.


Prefix: H=host G=guest.  
generally follow https://wiki.archlinux.org/title/Kernel/Traditional_compilation
* H: obtain kernel source
* G: copy out `/proc/config.gz`  (zcat to `<source>/.config`)
* H: `make nconfig (or menuconfig)` and make necessary changes.  
  suggestions below:
  * General setup -> Local version: `CONFIG_LOCALVERSION=-gdb`  
    **(you should do this to avoid making a kernel with the same <uname -r>)**
  * Kernel hacking
    * Compile-time checks and compiler options
      * Provide GDB scripts for kernel debugging: `CONFIG_GDB_SCRIPTS=y`
      * Generate readable assembler code: `CONFIG_READABLE_ASM=y`
    * x86 Debugging -> Choose kernel unwinder -> Frame pointer unwinder: `CONFIG_UNWINDER_FRAME_POINTER=y`
* H/G: make sure to use `modprobed-db` to reduce unnecessary compilation
  * G: `modprobed-db list > <source>/needed_mods`
  * H: `make LSMOD=needed_mods localmodconfig`
* H: `make` or `make -j<number of threads>`
* G: `cp arch/x86/boot/bzImage /boot/vmlinuz_linux_gdb`  
  (suggest not to use dash but instead underscore so grub doens't automatically pick up your kernel)  
  (skip if you want to use direct kernel boot. It seems grub does not support `vmlinux`, wtf???)
* G: `make modules_install`
* G: `mkinitcpio -k <new uname -r> -g /boot/initramfs_linux_gdb.img`  
  (use the `-k` option matching your new kernel's `uname -r`, this has to match the module directory `/lib/modules/<uname -r>`)
* G: `rm -rf /lib/modules/<old uname -r>` as needed

some helpers
* H: `make compile_commands.json`
* H: `make cscope`
* G: `systemctl enable serial-getty@ttyS0.service`

## Add a boot entry on guest

* open `/boot/grub/grub.cfg` and find your default boot menuentry, copy to `/etc/grub.d/40_custom`, and then modify the `linux` and `initrd` accordingly.  
  (skip if direct kernel boot)
  * suggest `linux /vmlinuz_linux_gdb root=UUID=<your root's uuid> rw loglevel=3 console=ttyS0,115200 nokaslr`
  * optionally add `GRUB_TERMINAL_OUTPUT="console vga_text"` in `/etc/default/grub` so that grub menu also works on tty
  * Finally `grub-mkconfig -o /boot/grub/grub.cfg`
* shutdown and launch with `virsh start <vm> --console`

## direct kernel boot 

(With virtiofs this seems to be a less attractive option)

Copy out to host the new initramfs.

Adjust xml as follows, make sure to include `nokaslr` if you want gdb to work
```xml
...
  <os>
    ...
    <kernel>/path/to/vmlinux</kernel>
    <initrd>/path/to/initramfs_linux_gdb.img</initrd>
    ...
  </os>
...
```

## GDB

Add to xml the following:
```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
...
<qemu:commandline>        
  <qemu:arg value='-s'/>  
</qemu:commandline>
```

start GDB:
* `gdb <source>/vmlinux`
* `target remote :1234`
