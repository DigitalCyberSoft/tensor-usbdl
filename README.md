# Tensor-USBDL

**Version:** v0.1.0
**Developer:** JoshuaDoes
**License:** GNU Affero General Public License v3

## Overview

Tensor-USBDL (Tensor USB Downloader) is a specialized tool for unbricking and bootloading Google Pixel devices with Tensor (Exynos-based) processors. It communicates with devices in "Exynos USB Boot" (EUB) mode and facilitates the upload of bootloader images via serial USB connections.

### Supported Devices

- **Tensor G1**: Pixel 6, Pixel 6a, Pixel 6 Pro (GS101)
- **Tensor G2**: Pixel 7, Pixel 7a, Pixel 7 Pro (GS201)
- **Tensor G3**: Pixel 8 series (GS301/Exynos9865) - theoretical support

## Getting Bootloader Images

Download the required bootloader images for your device:

### Tensor G2 / G3 (Pixel 7 & 8 series)

Download from the upstream release:
https://github.com/JoshuaDoes/tensor-usbdl/releases/tag/010

### Tensor G1 (Pixel 6 series)

GS101 images are available from this fork:
https://github.com/mkg20001/tensor-usbdl

### Directory Structure

Extract images to a `sources` directory matching your device:

```
sources/
├── gs201/           # Pixel 7, 7a, 7 Pro
│   ├── bl1.img
│   ├── pbl.img
│   ├── bl2.img
│   ├── gsa.img
│   ├── abl.img
│   ├── tzsw.img
│   ├── ldfw.img
│   └── bl31.img
│
└── gs301/           # Pixel 8 series
    ├── husky/       # Pixel 8 Pro
    │   └── *.img
    └── shiba/       # Pixel 8
        └── *.img
```

### Required Image Files

| File | Description | Required |
|------|-------------|----------|
| `bl1.img` | Primary bootloader (BL1) | Yes |
| `pbl.img` | Pre-bootloader (EPBL) | Yes |
| `bl2.img` | Secondary bootloader (BL2/BL2B) | Yes |
| `gsa.img` | Google Security Anchor (GSA1) | Yes |
| `abl.img` | Android Bootloader (ABL/ABLB) | Yes |
| `tzsw.img` | TrustZone Software (TZSW/TZSB) | Yes |
| `ldfw.img` | Loadable Firmware (LDFW/LDFB) | Yes |
| `bl31.img` | ARM Trusted Firmware EL3 (BL31/BL3B) | Yes |
| `gcf.img` | Generic Crypto Framework (GCF/GCFB) | Optional |
| `gsaf.img` | GSA Alternate | Optional |

### Device Codenames

| SoC | Device | Codename |
|-----|--------|----------|
| GS101 | Pixel 6 | oriole |
| GS101 | Pixel 6a | bluejay |
| GS101 | Pixel 6 Pro | raven |
| GS201 | Pixel 7 | panther |
| GS201 | Pixel 7a | lynx |
| GS201 | Pixel 7 Pro | cheetah |
| GS301 | Pixel 8 | shiba |
| GS301 | Pixel 8 Pro | husky |

**Note:** Images are device-specific. Using images from a different Tensor generation will not work.

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                        Recovery Flow                             │
├─────────────────────────────────────────────────────────────────┤
│  1. Device enters EUB mode (bricked or manual trigger)          │
│  2. Tool detects device via USB (VID:18D1 PID:4F00)             │
│  3. Serial connection established at 115200 baud                │
│  4. Device requests bootloaders one by one                      │
│  5. Tool sends: BL1 → DPM → EPBL → BL2 → ABL → TZSW → etc.     │
│  6. Device boots to Fastboot mode                               │
│  7. Tool monitors battery charging until safe to boot           │
│  8. Device reboots or returns to EUB for retry                  │
└─────────────────────────────────────────────────────────────────┘
```

## Project Structure

| File | Purpose |
|------|---------|
| `cmd/tensor-usbdl/main.go` | Main CLI application and state machine |
| `cmd/tensor-usbdl/utils.go` | File I/O and checksum utilities |
| `dnw.go` | DNW protocol handler for EUB serial communication |
| `fastboot.go` | Fastboot USB protocol implementation |
| `fastboot_stub.go` | Stub for builds without fastboot (`-tags nofastboot`) |
| `message.go` | Protocol message parser |
| `command.go` | DNW command packet serializer |
| `devices.go` | USB device enumeration |

## Fork Improvements

This fork includes several enhancements over the upstream version:

### Battery Monitoring & Safety
- Real-time battery voltage and current display
- Formatted output with comma separators (e.g., `4,302mV @ 1,123mA`)
- Charging rate assessment with warnings for slow charging
- Detection of discharging state with clear replug instructions
- Automatic monitoring loop during Fastboot mode

### USB Speed Display
- Human-readable USB connection speed
- Shows maximum charging capability per USB spec:
  - USB 2.0 Full Speed (12 Mbps, max 500mA)
  - USB 2.0 High Speed (480 Mbps, max 500mA)
  - USB 3.0 SuperSpeed (5 Gbps, max 900mA)
  - USB 3.1 SuperSpeed+ (10 Gbps, max 900mA)

### Fastboot Integration
- Query device variables (product, serial, bootloader version)
- Battery voltage/current monitoring via `getvar`
- OEM command support for GSC queries
- Device state detection (locked/unlocked/error)
- Automatic unlock attempt when unbricking succeeds
- Power management (reboot, powerdown)

### Reliability Improvements
- Proper OKAY/FAIL response handling for all fastboot commands
- Retry logic for transient I/O errors
- Graceful device disconnection handling
- Reduced EUB scan interval (30 seconds)

## Bootloader Image Format

Files contain a 4096-byte header followed by the bootloader payload:

```
Offset    | Size   | Field
----------|--------|----------------------------------
0x000     | 512    | Unknown/model-specific
0x200     | 512    | Device series identifier
0x400     | 4      | Magic number
0x404     | 8      | Unknown
0x40C     | 4      | Body length (uint32)
0x410     | 4      | Flags ("USB Bootable" bit)
0x414     | 12     | Unknown/padding
0x420     | 32     | Signature 1
0x440     | 32     | Signature 2
0x460     | 2976   | Padding (zeros)
```

## Supported Bootloaders

| Name | Description | Transfer Mode |
|------|-------------|---------------|
| BL1/EPBL | Primary bootloader | Full |
| DPM | Device Performance Monitor | Full (4KB) |
| BL2/BL2B | Secondary bootloader | Header + Body |
| GSA/GSA1 | Google Security Processor | Full |
| ABL/ABLB | ARM Trusted Firmware | Header + Body |
| TZSW/TZSB | TrustZone firmware | Header + Body |
| LDFW/LDFB | Loadable firmware | Header + Body |
| BL31/BL3B | EL3 secure monitor | Header + Body |
| GCF/GCFB | Generic Crypto Framework | Optional, Header + Body |
| GSAF | Alternate GSA variant | Optional |

## Usage

### Basic Recovery (EUB Mode)

```bash
tensor-usbdl --src /path/to/bootloaders
```

### Monitor Fastboot Device

```bash
tensor-usbdl --fastboot
```

### Specify Device Serial

```bash
tensor-usbdl --fastboot --serial XXXXXX
```

### Key Flags

| Flag | Description |
|------|-------------|
| `--src`, `-i` | Source directory for bootloader images |
| `--factory`, `-f` | FBPK v2 bootloader image |
| `--fastboot` | Skip EUB, monitor fastboot device directly |
| `--serial` | Specify fastboot device serial |
| `--dnw` | Force DNW download address |
| `--stop` | Send DNW STOP command on connection |

## Dependencies

- `go.bug.st/serial` - Cross-platform serial port communication
- `github.com/google/gousb` - USB device communication (requires libusb)
- `github.com/spf13/pflag` - Command-line flag parsing
- `github.com/JoshuaDoes/crunchio` - Binary buffer manipulation
- `github.com/JoshuaDoes/logger` - Structured logging

### Building Without Fastboot (No libusb)

```bash
go build -tags nofastboot ./cmd/tensor-usbdl
```

## Protocol Details

### DNW Message Format

Messages are newline-delimited with colon-separated fields:

```
COMMAND:SUBCOMMAND:DEVICE_ID:ARGUMENT
```

Examples:
- `eub:req:09845001cddf16d00bd4:BL1` - Request for BL1
- `eub:ack:09845001:DPM` - Acknowledgment of DPM
- `exynos_usb_booting:Pixel6,6a,6Pro` - Device identification

### Fastboot Communication

Standard Google Fastboot protocol over USB bulk endpoints:
- VID: 0x18D1 (Google)
- PID: 0x4EE0 or 0xD00D (Fastboot mode)

## Commit History (Fork Improvements)

| Commit | Description |
|--------|-------------|
| 15e6239 | Remove automatic USB reset, show manual replug message |
| 5222fd5 | Fix comma formatting for negative numbers |
| fe6ce9e | Improve fastboot monitoring and battery status display |
| 7690639 | Add USB speed display and improve battery monitoring |
| e1444bb | Add OEM command support and improve fastboot monitoring |
| d4f4317 | Add fastboot monitoring and reliability improvements |

## Notes

- Device must have sufficient battery charge (~4200mV+) for stable boot
- If device is discharging, unplug and replug USB cable
- Use a USB-C port for faster charging (up to 900mA vs 500mA on USB-A)
- Full charge may take up to 24 hours on slow USB ports
