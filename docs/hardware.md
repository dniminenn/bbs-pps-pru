# Hardware Requirements

## Hardware

- **Board:** BeagleBone Black (AM335x, Debian Trixie, kernel 6.6.58-ti-rt-arm32)
- **GPS options tested:** u-blox NEO-6M, NEO-M8T (connected via UART4 P9_11 RX, P9_13 TX)
  - Note: NEO-M9N does not have a UART interface and is not suitable here
- **PPS pin:** P8_16 → GPIO1_14 → PRU0 R31 bit 14
- **RTC:** DS3231 on I2C2 (holdover)

## Prerequisites

### Kernel & Toolchain

```bash
# Confirm RT kernel
uname -r
# 6.6.58-ti-rt-arm32-r12

# PRU compiler (TI Code Generation Tools): should already be installed
ls /usr/share/ti/cgt-pru/bin/clpru

# Device tree compiler
apt install device-tree-compiler

# PRU Software Support Package (PSSP): clone from TI's git
# This provides headers (pru_cfg.h, pru_intc.h, pru_rpmsg.h, etc.)
# and the rpmsg_lib.lib needed to build the firmware.
git clone https://git.ti.com/git/pru-software-support-package/pru-software-support-package.git \
  /opt/source/pssp
```

The build expects these paths:
- **`PRU_CGT=/usr/share/ti/cgt-pru`**: TI PRU compiler toolchain (pre-installed on BeagleBone images)
- **`PSSP=/opt/source/pssp`**: PRU Software Support Package (cloned above)

[Next: Initial Setup & Overlays](setup.md) | [Previous: Architecture & Overview](architecture.md)
