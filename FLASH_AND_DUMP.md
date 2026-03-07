# ACE Pro GD32F303 — SWD Flash & Dump

Low-level flash and dump procedures for the ACE Pro MCU via SWD.
Use this when you need to recover a bricked device, restore a backup,
or flash firmware without Katapult.

> The GD32F303CCT6 responds identically to STM32F103 on SWD.
> Any STM32-compatible SWD debugger works.

> **Toolchain note:** `arm-none-eabi-gdb` must be in your `PATH`. If it is
> not installed system-wide, add your toolchain's `bin/` directory first:
> ```bash
> export PATH="/path/to/your/arm-none-eabi-toolchain/bin:$PATH"
> ```

## Flash Map

### Stock ACE firmware

| Region | Address | Size |
|--------|---------|------|
| Bootloader | `0x08000000` | 32 KiB |
| APP1 (running firmware) | `0x08008000` | 96 KiB |
| APP2 (OTA staging) | `0x08020000` | 96 KiB |
| OTA_DATA (update flag) | `0x0803E000` | 8 KiB |
| End of flash | `0x08040000` | — |

### After Katapult + Klipper (or patched stock ACE)

| Region | Address | Size |
|--------|---------|------|
| Katapult bootloader | `0x08000000` | 32 KiB (actual ~4 KiB, reserves 32 KiB) |
| Klipper or stock ACE app | `0x08008000` | 224 KiB |
| End of flash | `0x08040000` | — |

## Important

The ACE Pro firmware **disables the SWD debug port at startup**.
You must hold NRST low during the SWD connect sequence.

- **Black Magic Probe:** `monitor connect_rst enable` before scanning
- **ST-Link v2 (OpenOCD):** `reset_config srst_only srst_nogate connect_assert_srst`

---

## Black Magic Probe

### Dump full flash (256 KiB)

```bash
arm-none-eabi-gdb -batch \
  -ex "target extended-remote /dev/serial/by-id/usb-Black_Magic_Debug_*-if00" \
  -ex "monitor connect_rst enable" \
  -ex "monitor swd_scan" \
  -ex "attach 1" \
  -ex "dump binary memory ace_full_flash.bin 0x08000000 0x08040000" \
  -ex "detach"
```

### Flash an ELF (Katapult, Klipper)

```bash
arm-none-eabi-gdb -batch klipper.elf \
  -ex "target extended-remote /dev/serial/by-id/usb-Black_Magic_Debug_*-if00" \
  -ex "monitor connect_rst enable" \
  -ex "monitor swd_scan" \
  -ex "attach 1" \
  -ex "load" \
  -ex "compare-sections" \
  -ex "monitor reset run" \
  -ex "detach"
```

### Flash a raw .bin (restore backup)

GDB `load` requires an ELF. Wrap the `.bin` first:

```bash
# Convert .bin to ELF with correct load address
arm-none-eabi-objcopy -I binary -O elf32-littlearm \
  --rename-section .data=.text \
  --change-addresses 0x08000000 \
  ace_full_flash.bin ace_full_flash.elf

# Flash
arm-none-eabi-gdb -batch ace_full_flash.elf \
  -ex "target extended-remote /dev/serial/by-id/usb-Black_Magic_Debug_*-if00" \
  -ex "monitor connect_rst enable" \
  -ex "monitor swd_scan" \
  -ex "attach 1" \
  -ex "monitor erase_mass" \
  -ex "load" \
  -ex "compare-sections" \
  -ex "monitor reset run" \
  -ex "detach"
```

### Mass erase (wipe everything)

```bash
arm-none-eabi-gdb -batch \
  -ex "target extended-remote /dev/serial/by-id/usb-Black_Magic_Debug_*-if00" \
  -ex "monitor connect_rst enable" \
  -ex "monitor swd_scan" \
  -ex "attach 1" \
  -ex "monitor erase_mass" \
  -ex "detach"
```

---

## ST-Link v2 (via OpenOCD)

### OpenOCD config

Create `ace-gd32f303.cfg`:

```
source [find interface/stlink.cfg]
transport select hla_swd

set CHIPNAME gd32f303cct6
source [find target/stm32f1x.cfg]

reset_config srst_only srst_nogate connect_assert_srst
```

### Dump full flash (256 KiB)

```bash
openocd -f ace-gd32f303.cfg \
  -c "init" \
  -c "reset halt" \
  -c "flash read_bank 0 ace_full_flash.bin 0 0x40000" \
  -c "shutdown"
```

### Flash a .bin at specific address

```bash
# Flash Klipper at 0x08008000 (with Katapult bootloader, 32KiB offset)
openocd -f ace-gd32f303.cfg \
  -c "init" \
  -c "reset halt" \
  -c "flash write_image erase out/klipper.bin 0x08008000" \
  -c "verify_image out/klipper.bin 0x08008000" \
  -c "reset run" \
  -c "shutdown"
```

### Flash full backup (restore)

```bash
openocd -f ace-gd32f303.cfg \
  -c "init" \
  -c "reset halt" \
  -c "flash write_image erase ace_full_flash.bin 0x08000000" \
  -c "verify_image ace_full_flash.bin 0x08000000" \
  -c "reset run" \
  -c "shutdown"
```

### Mass erase

```bash
openocd -f ace-gd32f303.cfg \
  -c "init" \
  -c "reset halt" \
  -c "stm32f1x mass_erase 0" \
  -c "shutdown"
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| SWD scan fails / no target found | Firmware disables debug port at startup | Use `connect_rst enable` (BMP) or `connect_assert_srst` (OpenOCD) |
| `Writing to flash forbidden` | Used GDB `restore` instead of `load` | Convert `.bin` → ELF with `objcopy`, then use `load` |
| Flash verify mismatch after write | Stale flash data from previous firmware | Do `monitor erase_mass` (BMP) or `mass_erase` (OpenOCD) before writing |
| ST-Link can't connect | Wrong transport or reset config | Use `hla_swd` transport, ensure `connect_assert_srst` is set |
