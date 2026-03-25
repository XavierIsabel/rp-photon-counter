# rp-photon-counter

## What This Is
FPGA-based photon counter for the Red Pitaya STEMlab 125-14. Performs real-time pulse detection at 125 MSPS on the FPGA, with a TCP server on the RP's ARM core and a Python client on the host PC. Designed for SiPM/SPAD detectors (tested with Thorlabs PDA42 SiPM amplified detector). Used for Raman spectroscopy and low-light spectroscopy applications.

## Tech Stack
- **FPGA**: SystemVerilog on Xilinx Zynq Z7010 (Red Pitaya v0.94 base project)
- **Build**: Vivado 2020.1 WebPACK (installed at `/opt/Xilinx/Vivado/2020.1/`)
- **Server**: Python 3.10 (runs on Red Pitaya ARM Linux)
- **Client**: Python 3.12 + uv (runs on host PC), matplotlib for live plotting

## Hardware Setup
- **Board**: STEMlab 125-14 Pro v2.0 — identifies as `z10_125_pro_v2` (Z7010 FPGA, NOT Z7020)
- **Input**: IN1 SMA, **HV jumper** (right position, +-20V range). 1 LSB = 2.44 mV.
- **Network**: Direct Ethernet, static link-local IP `169.254.32.2`
- **SSH**: `ssh -i ~/.ssh/id_rp root@169.254.32.2` (key-based auth configured)
- **Detector**: Thorlabs PDA42 SiPM, outputs +-2V pulses, ~60-85 mV single-photon pulses in HV mode
- **Optimal threshold**: ~28 ADC units for single-photon discrimination (dark counts ~5k cps)

## Commands

### Build FPGA bitstream
```bash
source /opt/Xilinx/Vivado/2020.1/settings64.sh
cd ~/rp-photon-counter/RedPitaya-FPGA
make PRJ=v0.94 MODEL=Z10    # Z10, not Z20!
```
The `make` will fail at the `xsct` step (FSBL) — that's expected. The bitstream is built successfully.

### Convert bitstream
```bash
cd ~/rp-photon-counter/RedPitaya-FPGA/prj/v0.94/out
echo "all:{ red_pitaya.bit }" > red_pitaya.bif
bootgen -image red_pitaya.bif -arch zynq -process_bitstream bin -o red_pitaya.bit.bin -w
```

### Deploy to Red Pitaya
```bash
scp -i ~/.ssh/id_rp red_pitaya.bit.bin root@169.254.32.2:/root/photon_counter.bit.bin
ssh -i ~/.ssh/id_rp root@169.254.32.2 'mount -o rw,remount /opt/redpitaya && cp /root/photon_counter.bit.bin /opt/redpitaya/fpga/z10_125_pro_v2/v0.94/fpga.bit.bin && sync && mount -o ro,remount /opt/redpitaya && reboot'
```
**Always verify hash after deploy** — the SSH connection drops during reboot, which can silently truncate the copy.

### Start TCP server (on RP after each reboot)
```bash
ssh -i ~/.ssh/id_rp root@169.254.32.2 '/root/start_photon.sh'
```

### Run live monitor (on host PC)
```bash
cd ~/rp-photon-counter/client
uv run python3 live_monitor.py --threshold 28 --deadtime 16 --gate-ms 100 --stream-ms 200 --histogram
```

## Project Structure
```
rp-photon-counter/
  fpga/
    rtl/photon_counter.sv    # FPGA module: discriminator + counter + histogram
    apply_patch.sh           # Patches RP top module to wire in photon_counter on sys[7]
    patch_top_z20.md         # Manual patching instructions (Z20 variant)
  server/
    photon_server.py         # TCP server on RP ARM, memory-maps FPGA registers via /dev/mem
  client/
    photon_client.py         # Python client library (PhotonCounter class)
    live_monitor.py          # Real-time matplotlib plotting
    pyproject.toml           # uv project (matplotlib, numpy)
  test_devmem.sh             # Low-level register test via devmem (run on RP)
  RedPitaya-FPGA/            # Cloned RP FPGA repo (gitignored) — patched in prj/v0.94/rtl/
```

## Architecture

### FPGA Module (`photon_counter.sv`)
Sits on system bus slot `sys[7]` at base address `0x40700000`. Reads ADC channel 1 at 125 MSPS.

**Pulse detection**: Rising-edge threshold crossing (`adc[n] >= threshold AND adc[n-1] < threshold`), followed by configurable dead time to prevent re-triggering.

**Registers** (offset from `0x40700000`):
| Offset | Name | R/W | Description |
|--------|------|-----|-------------|
| 0x00 | CTRL | R/W | [0]=enable, [1]=reset (auto-clears) |
| 0x04 | THRESHOLD | R/W | 16-bit signed threshold |
| 0x08 | DEAD_TIME | R/W | Dead time in clock cycles (8 ns each) |
| 0x0C | COUNT | R | 32-bit cumulative pulse count |
| 0x10 | COUNT_RATE | R | Pulses in last gate period |
| 0x14 | GATE_PERIOD | R/W | Gate period in cycles (125M = 1 sec) |
| 0x18 | PEAK_LAST | R | ADC value at last threshold crossing |
| 0x1C | STATUS | R | [0]=enabled, [1]=overflow |
| 0x20 | ADC_RAW | R | Current ADC sample (for tuning) |
| 0x24 | HIST_SHIFT | R/W | Histogram bit shift (0-10) |
| 0x100-0x1FF | HIST[0..63] | R | 64-bin pulse height histogram |

### Data Flow
```
PDA42 SiPM --[SMA]--> RP ADC (125 MSPS, HV mode)
                            |
                       [FPGA: photon_counter on sys[7]]
                            |
                       AXI registers (0x40700000)
                            |
                       [ARM: photon_server.py, mmap /dev/mem]
                            |
                       TCP port 5555
                            |
                       [PC: photon_client.py / live_monitor.py]
```

### TCP Protocol
Text-based, one command per line, one response per line.
Key commands: `ENABLE`, `DISABLE`, `RESET`, `SET_THRESHOLD <val>`, `SET_DEADTIME <cycles>`, `SET_GATE <cycles>`, `SET_HIST_SHIFT <val>`, `GET_COUNT`, `GET_RATE`, `GET_ADC`, `GET_PEAK`, `GET_STATUS`, `GET_HISTOGRAM`, `GET_CONFIG`, `STREAM [interval_ms]`, `STOP`.

## Key Patterns

- **Bus protocol**: RP uses a custom `sys_bus_if` (not AXI4-Lite). Set `sys_ack` and `sys_rdata` in the same clock cycle as `sys_en`. Match the pattern in `red_pitaya_hk.v`.
- **Build model**: Board identifies as `z10_125_pro_v2` via `profiles -f`. Build with `MODEL=Z10`. The Z10 top module is `red_pitaya_top.sv` (not `_Z20`).
- **Filesystem**: `/opt/redpitaya` is VFAT mounted read-only. Use `mount -o rw,remount /opt/redpitaya` before writing, then remount read-only after.
- **Bitstream deploy**: Always verify `md5sum` after copying. SSH drops during `reboot` can silently truncate the copy.
- **FPGA size constraint**: Z7010 has 4400 slices. The v0.94 ecosystem + photon counter nearly fills it. Histogram is limited to 64 bins. No room for large additions.
- **ADC data**: 14-bit ADC stored in 16-bit signed. Upper 2 bits are sign-extension. Use `{2'b0, adc_dat_i[13:0]}` when storing as unsigned to avoid sign issues.

## What Still Needs Work
- [ ] Auto-start TCP server on boot (add to startup.sh or systemd)
- [ ] Restore original bitstream script (backup at `fpga.bit.bin.bak`)
- [ ] Peak tracking during dead time (removed due to signed comparison issues — currently captures value at threshold crossing only)
- [ ] LV mode support for smaller signals (needs signal attenuation to stay within +-1V)
- [ ] Multi-photon peak resolution in histogram (needs LV mode for adequate ADC resolution)
- [ ] Bias voltage control for SiPM optimization
