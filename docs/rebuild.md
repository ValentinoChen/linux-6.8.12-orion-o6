# Rebuild Linux 6.8.12 for Radxa Orion O6

This document records the minimal reproducible path for rebuilding the working
`6.8.12-o6test` kernel used on a Radxa Orion O6 (AArch64).

The working source tree was checked against the official kernel.org Linux 6.8.12
release. After removing build-generated files, a checksum-based bidirectional
comparison reported no differences. No Orion O6-specific source patch was found.

Therefore, a rebuild can start from the official upstream Linux 6.8.12 source
release. The repository does not need to carry a full copy of the Linux source
tree.

---

## 1. Reproducibility inputs

Required inputs:

- Official Linux 6.8.12 source from kernel.org
- `configs/config-6.8.12-o6test`
- The boot parameters recorded below
- RTL8126 out-of-tree DKMS driver, tested as `r8126-dkms 10.015.00-2`
- A working EFI System Partition using systemd-boot / BLS entries

Known final kernel release:

```text
6.8.12-o6test
```

Known SHA256 of the final kernel configuration:

```text
2cea27337de764c9b455cdde1bda04fc1df81890e6491f91a57aa40fbd002510e
```

Known SHA256 of the official `linux-6.8.12.tar.xz` used for comparison:

```text
19b31956d229b5b9ca5671fa1c74320179682a3d8d00fc86794114b21da86039
```

---

## 2. Obtain the upstream source

Example:

```bash
mkdir -p ~/kernel-work
cd ~/kernel-work

wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.8.12.tar.xz
sha256sum linux-6.8.12.tar.xz
```

The checksum must match:

```text
19b31956d229b5b9ca5671fa1c74320179682a3d8d00fc86794114b21da86039
```

Extract it:

```bash
tar -xf linux-6.8.12.tar.xz
cd linux-6.8.12
```

Confirm the version:

```bash
head -n 6 Makefile
```

Expected version fields include:

```text
VERSION = 6
PATCHLEVEL = 8
SUBLEVEL = 12
EXTRAVERSION =
```

---

## 3. Restore the tested kernel configuration

Copy the configuration stored in this repository into the source tree:

```bash
cp /path/to/linux-6.8.12-orion-o6/configs/config-6.8.12-o6test .config
```

Verify the copied file before building:

```bash
sha256sum .config
```

Expected:

```text
2cea27337de764c9b455cdde1bda04fc1df81890e6491f91a57aa40fbd002510e
```

The saved configuration already contains:

```text
CONFIG_LOCALVERSION="-o6test"
# CONFIG_LOCALVERSION_AUTO is not set
```

so the expected installed kernel release is:

```text
6.8.12-o6test
```

For the exact same Linux 6.8.12 source release, the saved `.config` is the
authoritative configuration artifact. If `make olddefconfig` is run for any
reason, re-check the resulting configuration before treating it as identical to
the archived build.

---

## 4. Build natively on the Orion O6

The original successful environment was AArch64. A native build can be started
with:

```bash
make ARCH=arm64 -j"$(nproc)" Image modules
```

Useful output:

```text
arch/arm64/boot/Image
```

Install modules:

```bash
sudo make ARCH=arm64 modules_install
sudo depmod -a 6.8.12-o6test
```

Verify:

```bash
ls -ld /lib/modules/6.8.12-o6test
```

### Optional x86_64 cross-build

A cross-build is possible in principle with an AArch64 toolchain, for example:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j"$(nproc)" Image modules
```

This cross-build path was not the validated deployment path for this project.
Module installation, initramfs generation, EFI deployment, and DKMS rebuilding
must still be performed against the target Orion O6 root filesystem.

---

## 5. Install the kernel image

The tested EFI layout uses:

```text
/boot/efi/RadxaOS/6.8.12-o6test/
```

Create the directory if necessary:

```bash
sudo mkdir -p /boot/efi/RadxaOS/6.8.12-o6test
```

Copy the ARM64 kernel image:

```bash
sudo cp arch/arm64/boot/Image \
  /boot/efi/RadxaOS/6.8.12-o6test/linux
```

Before copying large files, check EFI free space:

```bash
df -h /boot/efi
```

The tested system used a roughly 1 GiB EFI System Partition and it was already
close to full capacity.

---

## 6. Generate and install the initramfs

On a Debian/RadxaOS-style target, after installing the kernel modules:

```bash
sudo update-initramfs -c -k 6.8.12-o6test
```

Confirm:

```bash
ls -lh /boot/initrd.img-6.8.12-o6test
```

Copy it to the tested EFI layout:

```bash
sudo cp /boot/initrd.img-6.8.12-o6test \
  /boot/efi/RadxaOS/6.8.12-o6test/initrd.img-6.8.12-o6test
```

If the target distribution uses a different initramfs tool, use the
distribution's supported method instead.

---

## 7. Create the systemd-boot / BLS entry

Create:

```text
/boot/efi/loader/entries/RadxaOS-6.8.12-o6test.conf
```

The tested entry was:

```text
title Linux 6.8.12-o6test (TEST)
version 6.8.12-o6test
sort-key zz-o6test
options root=UUID=<TARGET_ROOT_UUID> console=ttyAMA0,115200n8 earlycon=pl011,0x040d0000 acpi=force loglevel=8 rw earlycon consoleblank=0 console=tty1 coherent_pool=2M irqchip.gicv3_pseudo_nmi=0 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory swapaccount=1 nokaslr kasan=off mte.mode=sync
linux /RadxaOS/6.8.12-o6test/linux
initrd /RadxaOS/6.8.12-o6test/initrd.img-6.8.12-o6test
```

Do **not** reuse the root filesystem UUID from another disk.

Find the target root filesystem UUID with, for example:

```bash
findmnt /
sudo blkid
```

Replace `<TARGET_ROOT_UUID>` with the UUID of the target root filesystem.

The important console/debug settings used for bring-up were:

```text
console=ttyAMA0,115200n8
earlycon=pl011,0x040d0000
loglevel=8
consoleblank=0
console=tty1
```

For the first reboot, keep a previously known-good kernel as the default and
select `6.8.12-o6test` manually. Change the default only after validation.

---

## 8. Restore RTL8126 Ethernet support

The Orion O6 system used two Realtek RTL8126 5GbE controllers:

```text
PCI vendor/device: 10ec:8126
```

The upstream Linux 6.8.12 `r8169` driver did not provide the required device
alias in this setup. Ethernet was restored with the out-of-tree DKMS driver:

```text
r8126-dkms 10.015.00-2
module version: 10.015.00-NAPI
```

After the matching DKMS source/package is available on the target, install it
for `6.8.12-o6test`. A typical DKMS command is:

```bash
sudo /usr/sbin/dkms install -m r8126 -v 10.015.00-2 -k 6.8.12-o6test
```

Then:

```bash
sudo depmod -a 6.8.12-o6test
sudo /usr/sbin/modprobe r8126
```

Verify:

```bash
/usr/sbin/dkms status
modinfo r8126 | grep -E 'version|8126|vermagic'
lspci -nnk -d 10ec:8126
ip -br link
```

Expected PCI state:

```text
Kernel driver in use: r8126
Kernel modules: r8126
```

On a fresh disk, do not depend on the RTL8126 interface to download its own
driver. Keep the DKMS package/source available locally, or have another
temporary network path.

---

## 9. First-boot validation

After booting the test entry:

```bash
uname -r
```

Expected:

```text
6.8.12-o6test
```

Check:

```bash
cat /proc/cmdline
lspci -nnk -d 10ec:8126
ip -br addr
/usr/sbin/dkms status
```

If serial console access is available, use:

```text
115200 baud
ttyAMA0
```

The validated setup produced both UART kernel logs and a local `tty1` console.

---

## 10. Original tested root UUID

For historical reference, the original working system used:

```text
ce95976e-7c04-4b82-8251-c780b6b3fac1
```

This UUID belongs to that specific disk installation and must not be copied
blindly to a new disk.

---

## 11. Upstream source verification record

The working source tree was compared against a freshly extracted copy of the
official kernel.org Linux 6.8.12 tarball.

The tarball checksum was verified as:

```text
19b31956d229b5b9ca5671fa1c74320179682a3d8d00fc86794114b21da86039
```

A disposable copy of the working tree was cleaned with:

```bash
make ARCH=arm64 mrproper
```

One remaining generated file identified itself as automatically generated:

```text
security/selinux/av_permissions.h
```

After removing that generated comparison artifact from the disposable copy, the
final checksum-based bidirectional comparison was:

```bash
rsync -rcln --delete --itemize-changes \
  /path/to/linux-6.8.12-clean/ \
  /path/to/linux-6.8.12-work-cleaned/
```

It produced no output.

Conclusion:

```text
No modification to the upstream Linux 6.8.12 source tree was found.
No Orion O6-specific kernel source patch is required for this tested build.
```

The important reproducibility artifacts are therefore the final kernel
configuration, boot configuration, and the external RTL8126 DKMS driver rather
than a duplicate copy of the upstream Linux source tree.

---

## 12. Scope and limitations

This document reproduces the kernel bring-up state that was actually validated
on the Orion O6. It is not a general installation guide for every Orion O6
firmware, disk layout, or Linux distribution.

In particular:

- the root filesystem UUID is disk-specific;
- EFI free space must be checked before installation;
- initramfs commands may differ by distribution;
- RTL8126 support depends on an external DKMS driver;
- the optional x86_64-to-AArch64 cross-build path was not the validated build
  path for this project.
