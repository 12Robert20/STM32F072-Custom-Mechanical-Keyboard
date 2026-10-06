# STM32F072 Custom Mechanical Keyboard Controller

A custom hardware controller and switch matrix PCB designed in KiCad, powered by the STMicroelectronics STM32F072CBU6 microcontroller. Built for standard ANSI/ISO mechanical keyboard configurations with full N-Key Rollover (NKRO), native USB Type-C connectivity, and hardware-level ESD protection.

## 1. Project Overview

This project implements a self-contained mechanical keyboard controller and matrix layout. The architecture was developed to prioritize reliability, low Bill-of-Materials (BOM) complexity, and robust electrical protection without requiring external crystals or SPI flash memories.

### Key Specifications

* **Microcontroller:** STM32F072CBU6 (Arm Cortex-M0, 48 MHz, UFQFPN-48 package)
* **Switch Matrix:** 6 rows $\times$ 17 columns (up to 102 switch positions)
* **Anti-Ghosting:** Individual 0603 switching diodes per key (Full NKRO)
* **USB Interface:** 16-pin USB Type-C with dual $5.1\ \text{k}\Omega$ CC pulldowns (UFP / Device mode)
* **ESD Protection:** STMicroelectronics USBLC6-2SC6 low-capacitance TVS diode array on USB $D+$ and $D-$ lines
* **Power Regulation:** Diodes Inc. AP2112K-3.3 LDO (3.3V, 600 mA output, high PSRR) with ceramic decoupling
* **Programming & Debug:** 4-pin SWD interface (ST-Link compatible) plus hardware Reset and Boot0 pushbuttons
* **Status Indicators:** 3 onboard 1206 LEDs for Caps Lock, Num Lock, and Scroll Lock
* **Impedance Control:** Controlled $90\ \Omega$ differential impedance on USB 2.0 Full-Speed lines

## 2. Architecture & Design Rationale

During initial schematic capture and floorplanning, three controller platforms were evaluated:

| Microcontroller | USB Architecture | External Flash Required | External Oscillator Required | Design Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **RP2040** | USB 2.0 FS | Yes (External QSPI NOR) | Yes (12 MHz Crystal) | Higher BOM count and routing density |
| **CH552** | USB 2.0 FS | No (Internal) | No (Internal RC) | Low flash/RAM headroom, limited GPIO drive |
| **STM32F072CBU6** | USB 2.0 FS (Native) | No (128 KB On-Chip) | No (Clock Recovery System) | **Selected:** Clean single-chip layout, native DFU, robust tooling |

The STM32F072 utilizes internal Clock Recovery System (CRS) hardware to synchronize its internal 48 MHz high-speed oscillator (HSI48) directly to USB Start-of-Frame (SOF) packets. This eliminates the need for an external quartz crystal or resonator, saving board real estate and reducing susceptibility to mechanical shock.

## 3. Hardware Subsystems

### 3.1. Matrix Architecture (6 x 17)

The board routes 6 row lines (`ROW_0` through `ROW_5`) and 17 column lines (`COLS_0` through `COLS_16`).

* Standard keyboard rows: Function row, Number row, QWERTY, Home row, Bottom row, and Space/Modifier row.
* Each switch footprint is paired with a surface-mount 0603 diode oriented from row to column to prevent back-feeding and ghosting during simultaneous multi-key presses.

### 3.2. Power Supply (VBUS to +3.3V)

* **Input:** $5\ \text{V}$ nominal supplied via USB VBUS.
* **Regulation:** AP2112K-3.3 SOT-23-5 low-dropout linear regulator capable of supplying up to $600\ \text{mA}$ with low quiescent current and drop-out voltage of $\sim 250\ \text{mV}$ at full load.
* **Decoupling:** $1\ \mu\text{F}$ input and output MLCC ceramic capacitors supplemented by local $100\ \text{nF}$ decoupling caps adjacent to all MCU $V_{DD}$ pins.

### 3.3. USB-C Entry and Protection

* **Connector:** 16-pin USB Type-C receptacle (USB 2.0 signal lines only).
* **Configuration Channel:** Two independent $5.1\ \text{k}\Omega$ ($1\%$) pulldown resistors tied to `CC1` and `CC2` to request $5\ \text{V}$ at up to $3\ \text{A}$ from standard Type-C and Type-C PD source devices.
* **Transient Voltage Suppression (TVS):** An STMicroelectronics `USBLC6-2SC6` array placed directly adjacent to the USB receptacle clamps electrostatic discharge events up to $15\ \text{kV}$ air / $8\ \text{kV}$ contact without degrading $12\ \text{Mbps}$ Full-Speed differential signal integrity ($C_{I/O} < 3.5\ \text{pF}$).

### 3.4. Status Indicators

* 3 GPIO-driven indicator channels (`CAPS_LED`, `NUM_LED`, `SCRL_LED`).
* Output lines feature dedicated $200\ \Omega$ series current-limiting resistors paired with 1206 package status LEDs.

## 4. PCB Layout and Differential Routing

The PCB is implemented on a standard 2-layer FR-4 substrate ($1.6\ \text{mm}$ thickness):

* **USB Differential Pair:** Calculated and routed for a $90\ \Omega \pm 10\%$ differential impedance target ($Z_{\text{diff}}$):
  * Trace width: $0.28\ \text{mm}$
  * Differential spacing: $0.20\ \text{mm}$
  * An unbroken ground plane on Layer 2 directly references the pair from the receptacle through the TVS diode up to MCU pins `PA11` ($D-$) and `PA12` ($D+$).
* **Ground Return Paths:** Top and bottom layers utilize copper fills tied together with regular via stitching to minimize loop area across switch matrix column returns.
* **Mechanical Tolerances:** Solder mask expansions and component keep-outs are verified against mechanical plate cuts and switch housing footprints.

## 5. Firmware Support

The hardware is compatible with several embedded keyboard firmware ecosystems:

1. **QMK Firmware:** Using the ChibiOS HAL for STM32F072 targets (`STM32F072xB`).
2. **ZMK Firmware:** Via custom Zephyr board definitions.
3. **Bare-metal STM32Cube / Custom C:** Direct USB HID implementation using standard ST USB Device Middleware.

### Programming & Flashing

* **Initial Flashing (SWD):** Connect an ST-Link V2/V3 programmer to header `J2`:
  * Pin 1: `+3.3V`
  * Pin 2: `SWDIO` (PA13)
  * Pin 3: `SWCLK` (PA14)
  * Pin 4: `GND`
* **USB DFU Bootloader:** Hold down the onboard `BOOT0` button and toggle `RESET` to enter the internal ST factory system memory bootloader directly over USB Type-C.

## 6. Fabrication & Assembly Notes

* **Layer Count:** 2 layers
* **Minimum Trace / Space:** 0.2 mm / 0.2 mm
* **Minimum Via Size:** 0.3 mm drill / 0.6 mm annular ring
* **Surface Finish:** ENIG recommended (or Lead-free HASL)
* **Passive Packages:** 0603 (diode array and decoupling) / 1206 (indicator LEDs)
* **SMD IC Footprint:** UFQFPN-48 with central exposed thermal ground pad

## 7. Author & Inquiries

Designed by an independent hardware engineer specializing in KiCad PCB layout, embedded hardware design, and DFM optimization.

Open for freelance inquiries regarding schematic capture, board redesigns, and layout for prototype or production hardware. Direct messages can be sent via LinkedIn or GitHub issues.