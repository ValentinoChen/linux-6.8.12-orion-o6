# Radxa Orion O6 Linux 6.8.12-o6test Bring-up Notes

## 1. Final kernel configuration

### 1.1 Final kernel release

The validated kernel reports:

```text
Linux RadxaOrionO6 6.8.12-o6test ... aarch64 GNU/Linux
```

Kernel release:

```text
6.8.12-o6test
```

The build tree is:

```text
/home/cyf/kernel-work/linux-6.8.12
```

The final config exists at:

```text
/home/cyf/kernel-work/linux-6.8.12/.config
```

and the installed copy is:

```text
/boot/config-6.8.12-o6test
```

These two files were verified to be byte-identical.

SHA256:

```text
2cea27337de764c9b455cdde1bda04fc1df81890e6491f91a57aa40fbd002510e
```

### 1.2 What was actually changed

The final configuration contains:

```text
CONFIG_LOCALVERSION="-o6test"
# CONFIG_LOCALVERSION_AUTO is not set
```

This produces the final kernel release string:

```text
6.8.12-o6test
```

A later comparison between:

```text
.config.old
.config
```

showed only:

```text
-SYSTEM_REVOCATION_KEYS ""
```

The two files already contained the same:

```text
CONFIG_LOCALVERSION="-o6test"
```

Therefore, the `LOCALVERSION` change occurred before `.config.old` was created.

### 1.3 About the 418 `diffconfig` lines

The command:

```bash
./scripts/diffconfig /boot/config-6.6.89-3-mte /boot/config-6.8.12-o6test | wc -l
```

returned:

```text
418
```

This must **not** be interpreted as 418 manual edits.

The comparison is between final configs from two different kernel versions. The result includes Kconfig changes caused by:

- options added or removed between Linux 6.6.89 and Linux 6.8.12;
- renamed symbols;
- dependency changes;
- changed defaults;
- automatic Kconfig normalization.

The complete early manual-edit history could not be reconstructed because the source directory does not contain Git history and the login shell did not provide usable command history.

For reproducibility, the **final validated `.config`** should therefore be treated as the authoritative configuration artifact.

---

## 2. EFI / systemd-boot configuration

### 2.1 EFI partition

The EFI System Partition is mounted at:

```text
/boot/efi
```

Device:

```text
/dev/sda2
```

Filesystem:

```text
vfat
```

Linux root filesystem:

```text
/dev/sda3
```

### 2.2 Linux 6.8.12 boot entry

Boot Loader Specification entry:

```text
/boot/efi/loader/entries/RadxaOS-6.8.12-o6test.conf
```

Content:

```text
title Linux 6.8.12-o6test (TEST)
version 6.8.12-o6test
sort-key zz-o6test
options root=UUID=ce95976e-7c04-4b82-8251-c780b6b3fac1 console=ttyAMA0,115200n8 earlycon=pl011,0x040d0000 acpi=force loglevel=8 rw earlycon consoleblank=0 console=tty1 coherent_pool=2M irqchip.gicv3_pseudo_nmi=0 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory swapaccount=1 nokaslr kasan=off mte.mode=sync
linux /RadxaOS/6.8.12-o6test/linux
initrd /RadxaOS/6.8.12-o6test/initrd.img-6.8.12-o6test
```

Compared with the previous `6.6.89-3-mte` boot entry, the main console/debug related changes were:

```text
quiet            removed
splash           removed
loglevel=4       -> loglevel=8
```

The following remained important for console output:

```text
console=ttyAMA0,115200n8
earlycon=pl011,0x040d0000
earlycon
consoleblank=0
console=tty1
```

This allows both:

- UART console output through `ttyAMA0`;
- local display console output through `tty1`.

### 2.3 Current default boot entry

`/boot/efi/loader/loader.conf` contains:

```text
default RadxaOS-6.6.89-3-llvm.conf
timeout 3
#console-mode keep
```

Therefore `6.8.12-o6test` is a separate validated test entry, while the default boot entry remains `6.6.89-3-llvm`.

---

## 3. RTL8126 network adapter support

### 3.1 Initial problem

Linux `6.8.12-o6test` booted successfully, but the two Realtek RTL8126 5GbE controllers were initially present on PCIe without the required working `r8126` driver.

The PCI devices are:

```text
01:00.0  Realtek RTL8126 5GbE Controller [10ec:8126]
31:00.0  Realtek RTL8126 5GbE Controller [10ec:8126]
```

The previous Linux 6.6.89 environment already contained an out-of-tree DKMS driver:

```text
r8126-dkms 10.015.00-2
```

### 3.2 Build and installation for Linux 6.8.12

The DKMS source was available under:

```text
/usr/src/r8126-10.015.00-2
```

The Linux 6.8.12 build tree was available through:

```text
/lib/modules/6.8.12-o6test/build
```

which pointed to:

```text
/home/cyf/kernel-work/linux-6.8.12
```

The driver was built for the new kernel with:

```bash
sudo /usr/sbin/dkms build   -m r8126   -v 10.015.00-2   -k 6.8.12-o6test
```

Then installed with:

```bash
sudo /usr/sbin/dkms install   -m r8126   -v 10.015.00-2   -k 6.8.12-o6test
```

Module dependencies were regenerated:

```bash
sudo depmod -a 6.8.12-o6test
```

The module was then loaded:

```bash
sudo /usr/sbin/modprobe r8126
```

### 3.3 Installed module

Final module:

```text
/lib/modules/6.8.12-o6test/updates/dkms/r8126.ko.xz
```

Driver version:

```text
10.015.00-NAPI
```

`vermagic`:

```text
6.8.12-o6test SMP preempt mod_unload aarch64
```

Relevant PCI aliases include:

```text
pci:v000010ECd00008126sv*sd*bc*sc*i*
pci:v000010ECd00005000sv*sd*bc*sc*i*
```

`lspci -nnk -d 10ec:8126` confirmed both RTL8126 controllers are bound to:

```text
Kernel driver in use: r8126
Kernel modules: r8126
```

After reboot, the driver remained available and the active interface obtained network connectivity. Remote SSH access was successfully restored.

---

## 4. EFI partition capacity

### 4.1 Current capacity

At the time of validation:

```text
Filesystem  Size   Used   Avail  Use%  Mounted on
/dev/sda2   1022M  998M   25M    98%  /boot/efi
```

The size limit comes from `/boot/efi` being a separate FAT EFI System Partition, not from it being an ordinary directory on the root filesystem.

### 4.2 Kernel directories stored in the ESP

The following command was used:

```bash
sudo sh -c 'du -sh /boot/efi/RadxaOS/*'
```

Result:

```text
202M  /boot/efi/RadxaOS/6.6.89-3-base
202M  /boot/efi/RadxaOS/6.6.89-3-llvm
202M  /boot/efi/RadxaOS/6.6.89-3-memsafety-kasan
202M  /boot/efi/RadxaOS/6.6.89-3-mte
84M   /boot/efi/RadxaOS/6.6.89-3-sky1
83M   /boot/efi/RadxaOS/6.8.12-o6test
```

The `6.8.12-o6test` directory contains approximately:

```text
49M  initrd.img-6.8.12-o6test
35M  linux
```

Total:

```text
~83M
```

The ESP was therefore almost full mainly because several large Linux 6.6.89 boot sets were still stored simultaneously.

No optimization or deletion is part of this record; this section only documents the observed state.

---

## 5. Validated result

The final validated state is:

```text
Kernel:        6.8.12-o6test
Architecture:  aarch64
Board:         Radxa Orion O6
UART console:  ttyAMA0 @ 115200
Local console: tty1
NIC:           Realtek RTL8126
NIC driver:    r8126 10.015.00-NAPI
SSH:           working after network connectivity is available
Boot method:   systemd-boot / Boot Loader Specification entry
```

The kernel was reboot-tested successfully with the RTL8126 DKMS module still available.

---

## 6. Files that should be archived

At minimum, archive:

```text
configs/config-6.8.12-o6test
docs/porting-notes.md
```

The final config SHA256 should remain:

```text
2cea27337de764c9b455cdde1bda04fc1df81890e6491f91a57aa40fbd002510e
```

For a stricter binary-reproducibility archive, future work may additionally preserve:

- the exact Linux 6.8.12 source tree;
- any source patches;
- the built kernel image;
- the initrd;
- the exact `r8126` DKMS source package/version.

Those extra artifacts are not required merely to document the four key bring-up changes summarized above.
