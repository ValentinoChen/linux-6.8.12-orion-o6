# Linux 6.8.12 for Radxa Orion O6

This repository records a Linux 6.8.12 ARM64 kernel bring-up on the **Radxa Orion O6**.

The validated kernel release is:

```text
6.8.12-o6test
```

The system has been verified to:

- boot Linux 6.8.12-o6test successfully on the Orion O6;
- provide UART console output on `ttyAMA0`;
- provide local display console output on `tty1`;
- load the Realtek RTL8126 5GbE driver through DKMS;
- obtain network connectivity and support remote SSH access;
- survive reboot with the RTL8126 driver still available.

## Repository contents

Planned layout:

```text
.
├── README.md
├── docs/
│   └── porting-notes.md
└── configs/
    └── config-6.8.12-o6test
```

The detailed bring-up record is in:

- [`docs/porting-notes.md`](docs/porting-notes.md)

The final kernel configuration should be archived as:

- `configs/config-6.8.12-o6test`

## Important note about the kernel config

The final Linux 6.8.12 config differs from the previous Linux 6.6.89 reference config by 418 `diffconfig` lines.  
**This does not mean 418 options were manually changed.** Many differences are caused by Kconfig changes between kernel versions.

The final validated config is therefore treated as the authoritative reproducibility artifact.

## Status

This repository is intended to preserve the working Orion O6 / ARM64 Linux 6.8.12 setup and the key steps required to reproduce it.

The original systemd-boot default entry remains Linux `6.6.89-3-llvm`; `6.8.12-o6test` exists as a separate boot entry.
