# PRU PPS: BeagleBone Black PRU Hardware Timestamping → Chrony SHM

**A working implementation of PRU-based PPS timestamping feeding Chrony via SHM, achieving steady-state clock offsets in the nanoseconds.**

A complete guide to implementing a PRU-based hardware-timestamped PPS source on the AM335x BeagleBone Black, feeding nanosecond-precision timing into Chrony via shared memory.

```
  PRU-ICSS (edge capture)              Linux (servo / policy)
 ┌──────────────────────────┐    ┌───────────────────────────────┐
 │ GPS PPS → P8_16 → R31    │    │ pru_pps_shm (SCHED_FIFO:50)   │
 │ PRU0 polls rising edge   ├───►│ IEP↔wall calibration          │
 │ IEP latch (200 MHz)      │    │ NTP SHM unit 2                │
 │ DRAM0: { seq, iep_lo }   │    │ Chrony refclock → adjtimex    │
 └──────────────────────────┘    └───────────────────────────────┘
```

## Performance Results

By entirely bypassing the Linux GPIO IRQ pathway and capturing the pulse edge using the 200 MHz PRU Industrial Ethernet Peripheral (IEP) hardware timer, this implementation eliminates traditional interrupt latency jitter.

- **Daemon Precision:** Typical wall-time residuals of 100–800 ns vs the true PPS edge.
- **System Clock Offset:** Once the Chrony servo converges, `chronyc tracking` reports system time offsets in the **low nanosecond domain** (often under ±10 ns).
- **Jitter Profile:** The estimated standard deviation converges to ~1 ns, an order-of-magnitude improvement over standard `pps-gpio` which routinely shows 5–20 µs of offset dispersion.

Detailed tracking data and logs can be found in the [Measurements & Validation](docs/measurements.md) document.

## Documentation

1. **[Architecture & Overview](docs/architecture.md)**: What is a PRU, why use it for PPS, and the system architecture
2. **[Hardware Requirements](docs/hardware.md)**: Supported hardware, GPS modules, and prerequisites
3. **[Initial Setup & Overlays](docs/setup.md)**: `uEnv.txt` configuration and device tree overlays
4. **[PRU Firmware](docs/firmware.md)**: Compiling and installing the PRU microcontroller firmware
5. **[Userspace Daemon & Chrony](docs/daemon.md)**: Building the shared memory daemon and configuring Chrony
6. **[Verification & Troubleshooting](docs/verification.md)**: Checking system behavior, remoteproc state, and common issues

### Deep Dives

7. **[Timestamp Model](docs/timestamp-model.md)**: What exactly is captured, which edge, what defines t=0
8. **[Clock Domains & Synchronization](docs/clock-domains.md)**: IEP to wall correlation, memory mechanism, ordering guarantees
9. **[Measurements & Validation](docs/measurements.md)**: Expected performance, real-world tracking/statistics, and diagnostic guidance

## Source Code

- [Firmware (`firmware/`)](firmware/): PRU0 C source, resource table, INTC map, linker command
- [Daemon (`daemon/`)](daemon/): Userspace SHM bridge and systemd service
- [Overlays (`overlays/`)](overlays/): Device tree overlays for pinmux and rpmsg
- [Tools (`tools/`)](tools/): Log parsing and statistics helpers
