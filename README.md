# illumos for AArch64 bootstrap

This tree is the bootstrap for illumos on AArch64.  It contains sufficient
pieces, and build materials, to build a bootable disk that can be used for
development, of the AArch64 port.

Pieces are missing from here (and matching pieces from illumos), to make this
easy, if not pleasant, at the current time.

## Dependencies

- An illumos system
- A compilation environment, headers, compiler, etc.
- `sudo` (pkg:/security/sudo)
- `sgdisk` (pkg:/system/storage/gptfdisk)
- `mkisofs` (pkg:/media/cdrtools)
- `rsync` (pkg:/network/rsync)
- Hopefully nothing else I forgot

## Building

To build there are three-ish steps

1. `make download` -- Fetch all the other sources we need at their correct
   versions.  (By default this takes shallow-ish clones of big trees, feel
   free to replace them with full clones).
1. `make setup` -- Build all the prerequisites to building illumos
1. `make illumos` -- Build illumos, a bit weirdly right now (we don't use
   `nightly`).  The environment file is in `env/aarch64` in this directory, and is what gets
   used for bootstrapping.
1. `make disk` -- Build the disk image which you can give to `qemu(1)`
    (this will complain about partitions and stuff, ignore it).  This will
    also ask for your password, so if you just run `make disk` and let
    dependencies take over, that won't go great.

This is rough around the edges, particularly regarding `Makefile` dependencies
and spurious rebuilds.  Please do fix this.

## Booting

Take the `illumos-disk.img` you have built, and the `inetboot.bin` for your
platform (likely qemu) out of the proto area, and supply them to `qemu`

I use something like this (on another system at present, as modern qemu has
issues under illumos):

```
sudo qemu-system-aarch64 -nographic -machine virt-4.1 -m 2g -smp 1 -cpu cortex-a53 -kernel inetboot.bin -append "-D /virtio_mmio@a003c00" -netdev bridge,id=net0,br=virbr0 -device virtio-net-device,netdev=net0,mac=52:54:00:70:0a:e4 -device virtio-blk-device,drive=hd0 -drive file=$PWD/illumos-disk.img,format=raw,id=hd0,if=none
```

- `-nographic` -- serial console on stdout
- `-machine virt-4.1` -- the current target qemu machine
- `-m 2g` -- 2G of memory, more can never hurt
- `-smp 1` -- 1 CPU, again, more shouldn't hurt
- `-cpu cortex-a53` -- an appropriate CPU for the port
- `-kernel inetboot.bin` -- the inetboot.bin for qemu taken from the illumos
  build
- `-append "-D /virtio_mmio@a003c00"` -- tell inetboot where to boot from
- `-netdev bridge,id=net0,br=virbr0` -- bridged networking, linuxily
- `-device virtio-net-device,netdev=net0,mac=52:54:00:70:0a:e4` -- virtual
  NIC, `platmac0` in the system
`-device virtio-blk-device,drive=hd0` -- our disk
`-drive file=illumos-disk.img,format=raw,id=hd0,if=none` -- the illumos disk
  image you want to boot.

A convenient way to do this is just to take the entire `qemu-setup/`
directory.

You will see messages from the temporary booter that seem worrying, about
missing boot_archives and `vdev_probe`, these are, weirdly, specific to the
currently weird booting method.

Once you have booted you will see copious boot messages both to aid debugging
and because the emulation is slow and it helps to keep track that something is
still happening, these are hardwired in the source at present, absent a real
booter.

There are also lingering bugs around SMF that may or may not fail your boot.
A notable one is that svc:/system/rbac always times out, and often takes a
long time to do so, be patient.

Note also that we will rebuild the `boot_archive` and then reboot again, on
first boot.  This is because the `boot_archive` created in
`tools/build_disk.sh` is an insufficiently good fake.

## After you boot

`root` has no password (at all, rather than an empty password).

Find something to fix! Lots of things are missing are broken!  Many of them important!

The most notable things for fixing stuff are that we have at present no mdb(1)
or kmdb(1) or dtrace(8), which is unfortunate.

We build a cross gdb(1) to tide us over, which can be used to analyze core
dumps from the virtual machine (you can just mount your pool back onto a
development machine, etc), or to analyze the running kernel code directly
(connect to the gdb server).

`(gdb) target remote tcp::1234`

There is a `.gdbinit` in this directory which does useful things like load the
`inetboot` and `unix` for qemu and provide a `load-kernel-modules` command to
load the other modules currently present in the running kernel.  It is not
great, but it is _something_.

## See Also

* [IPD 24 Support for 64-bit ARM (AArch64)](https://github.com/illumos/ipd/blob/master/ipd/0024/README.md)
