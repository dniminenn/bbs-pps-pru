# Userspace Daemon & Chrony Configuration

## Userspace Daemon: `pru_pps_shm`

### How It Works

1. **Opens `/dev/rpmsg_pru30`**: sends a setup byte to register its endpoint with the PRU
2. **`/dev/mem` mmap**: maps PRU DRAM0 (`0x4A300000`) and IEP registers (`0x4A32E000`)
3. **Blocks on `poll()`**: waits for the PRU to send an rpmsg notification on each PPS edge (2 s timeout)
4. **IEP-wall calibration**: takes 10 back-to-back `clock_gettime(CLOCK_REALTIME)` + IEP read samples, keeps the tightest bracket to get the most accurate mapping between IEP ticks and wall time
5. **ns/tick estimation**: uses consecutive IEP deltas (should be ~200,011,500 ticks per second at ~5 ns/tick) with a 0.9/0.1 IIR filter
6. **Projects PPS wall time**: `pps_wall_ns = cal_wall + (pps_iep - cal_iep) * ns_per_tick`
7. **Writes NTP SHM**: using mode-1 count handshake (odd while writing, even when done)

This blocking approach uses zero CPU between PPS pulses, compared to the ~20,000 wakeups/s a polling loop would require.

### SHM Struct Layout

The NTP SHM struct uses 64-bit `time_t` (Y2038-safe on this Debian Trixie system even on 32-bit ARM):

| Offset | Field | Notes |
|--------|-------|-------|
| 0 | `mode` (int32) | Set to 1 |
| 4 | `count` (int32) | Handshake: odd=writing, even=done |
| 8 | `clockTimeStampSec` (int64) | Rounded UTC second the pulse represents |
| 16 | `clockTimeStampUSec` (int32) | Always 0 (PPS is on-second) |
| 24 | `receiveTimeStampSec` (int64) | Wall time when edge was detected |
| 32 | `receiveTimeStampUSec` (int32) | Sub-second µs |
| 36 | `leap` (int32) | 0 |
| 40 | `precision` (int32) | -29 (~2 ns) |
| 56 | `clockTimeStampNSec` (uint32) | Always 0 |
| 60 | `receiveTimeStampNSec` (uint32) | Sub-second ns |

Chrony uses `clockTS - receiveTS` as its raw offset sample, so `receiveTimeStamp` must be the best available wall-clock estimate of the actual edge time, without rounding.

### Full Source & Build

- [pru_pps_shm.c](../daemon/pru_pps_shm.c)

```bash
gcc -O2 -Wall -o /usr/local/bin/pru_pps_shm pru_pps_shm.c -lrt -lm
```

---

## Systemd Service

Create `/etc/systemd/system/pru-pps-shm.service` (or use the one provided: [`pru-pps-shm.service`](../daemon/pru-pps-shm.service)):

```bash
systemctl daemon-reload
systemctl enable --now pru-pps-shm
```

**Note:** `remoteproc1` is PRU0 (`4a334000.pru`). `remoteproc2` is PRU1. The `ExecStartPre` sleep gives the rpmsg channel time to initialize. When the daemon starts, it writes a setup byte to `/dev/rpmsg_pru30` which tells the PRU the ARM endpoint addresses to use for notifications.

---

## Chrony Configuration

Relevant section of `/etc/chrony/chrony.conf`:

```
sched_priority 80

# GPS NMEA via gpsd/SHM unit 0: used for coarse time and lock reference
refclock SHM 0 refid GPS precision 1e-1 delay 0.4 poll 2 trust noselect

# PRU PPS via SHM unit 2: primary disciplining source
refclock SHM 2 refid PPS precision 1e-9 poll 1 prefer trust lock GPS pps

makestep 0.5 -1
maxupdateskew 100.0
rtcsync
rtcdevice /dev/rtc0
```

`lock GPS` tells Chrony to only use the PPS refclock when the GPS SHM refclock (unit 0) is also valid. This is essential for correct UTC second assignment. `pps` enables the PPS refclock mode.

---

## PHC Discipline (`phc2sys`)

Since the PRU driver captures the PPS sequence to discipline `CLOCK_REALTIME` via Chrony, the PTP hardware clock (e.g., the AM335x CPTS at `/dev/ptp0`) must be synchronized in software from the system clock.

This is accomplished using the standard `phc2sys` daemon from the `linuxptp` package:

```bash
# Sync the CPTS PHC (/dev/ptp0) from the system clock (CLOCK_REALTIME)
phc2sys -s CLOCK_REALTIME -c /dev/ptp0 -w
```

By running this as a systemd service along with `ptp4l`, your CPTS hardware clock will tightly track the PRU-disciplined system clock, enabling your device to act as an accurate PTP Grandmaster on your network.

### Systemd Service Examples

You can deploy `phc2sys` and `ptp4l` using the following systemd units.

**1. `/etc/systemd/system/phc2sys.service`**
```ini
[Unit]
Description=Synchronize PTP hardware clock (PHC) to system clock
After=chronyd.service network.target ptp4l.service
Wants=ptp4l.service

[Service]
Type=simple
# -s: source clock (system clock)
# -c: destination clock (CPTS hardware clock)
# -w: wait for ptp4l to start
ExecStart=/usr/sbin/phc2sys -s CLOCK_REALTIME -c /dev/ptp0 -w
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**2. `/etc/systemd/system/ptp4l.service`**
```ini
[Unit]
Description=Precision Time Protocol (PTP) Grandmaster service
After=network.target

[Service]
Type=simple
# -i eth0: bind to your network interface
# -m: print messages to stdout/syslog
ExecStart=/usr/sbin/ptp4l -i eth0 -m
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

[Next: Verification & Troubleshooting](verification.md) | [Previous: PRU Firmware](firmware.md)
