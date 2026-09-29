# HomeBus Fan Controller Node

The **HomeBus Fan Controller Node** is an intelligent, solid-state AC speed controller designed for residential ceiling fans, exhaust fans, and inductive motor loads. 

Unlike traditional multi-tap capacitor regulators or noisy phase-triac dimmers, the HomeBus Fan Controller combines zero-crossing detection, galvanically isolated gate driving, solid-state MOSFET switching, real-time load current sensing, and temperature monitoring on the RS-422 bus.

---

## Circuit Architecture

```mermaid
flowchart TD
    subgraph MainsPower ["230V AC Mains Interface"]
        LINE_IN["AC Line In (L_in)"] --> FUSE["1A Slow-Blow Fuse\n(2010T1A250V)"]
        FUSE --> MOV["10D471K Varistor\n(Surge Clamping)"]
        MOV --> ZCD["H11AA1 AC Optocoupler\n(Zero-Crossing Detection)"]
        MOV --> CT["ZHT103 / HWCT-5A\n(Current Transformer)"]
        CT --> MOS_STAGE["KIA7N65H Power MOSFET\n(Solid-State AC Switch)"]
        MOS_STAGE --> LINE_OUT["Controlled AC Out (L_out)"]
        NEUTRAL["AC Neutral (N)"] --- ZCD
        NEUTRAL --- MOV
    end

    subgraph IsolationDrive ["Galvanic Gate Drive Isolation"]
        DCDC["Hi-Link B2415S-1WR3\n(Isolated 24V -> 15V DC)"] --> GATE_DRV["HCPL-3120 / VO3120\n(High-CMR Optocoupled Gate Driver)"]
        GATE_DRV -->|15V Isolated Gate Pulse| MOS_STAGE
    end

    subgraph CoreMCU ["Control & Telemetry Subsystem"]
        MCU["WCH CH32X033F8P6\n(32-bit QingKe RISC-V @ 48 MHz)"]
        ZCD -->|ZCD Sync Pulse (EXTI)| MCU
        MCU -->|PWM Gate Trigger| GATE_DRV
        CT -->|Burden Resistor + MCP6001 Op-Amp| MCU
        NTC["10kΩ NTC Thermistor\n(Heatsink Temp)"] --> MCU
        LED["WS2812B RGB Status LED"] <--- MCU
    end

    subgraph BusComm ["HomeBus Network Interface"]
        DIP["8-Bit DIP Switch\n(Hardware Node ID)"] --> SR["74HC165D Shift Register"] --> MCU
        XCVR["MAX491E Transceiver\n(RS-422 Full-Duplex)"] <--> MCU
        TVS["SM712 TVS Array"] <--> XCVR
    end
```

---

## Key Hardware Components

| Subsystem | Component | Package | Technical Details & Function |
| :--- | :--- | :--- | :--- |
| **Microcontroller** | WCH `CH32X033F8P6` | TSSOP-20 | 32-bit QingKe RISC-V core @ 48 MHz. Manages zero-crossing interrupt timing, phase-angle phase delay calculation, anti-stall algorithms, and current telemetry. |
| **Zero-Crossing Detector** | Fairchild/Vishay `H11AA1` | DIP-6 | Bidirectional AC-input optocoupler. Generates clean logic pulses at 100 Hz (50 Hz mains) to sync phase-angle timing with sub-microsecond precision. |
| **Isolated Gate Driver** | Broadcom `HCPL-3120` / `VO3120` | DIP-8 / SOP-8 | High-speed optocoupled MOSFET/IGBT gate driver with 2.5A peak output and high common-mode rejection (CMR > 15 kV/µs). Protects logic from mains transients. |
| **Isolated DC-DC** | Hi-Link `B2415S-1WR3` | SIP-4 | 1W galvanically isolated DC-DC converter (24VDC in to 15VDC out, 67mA). Supplies clean, isolated 15V power to the floating MOSFET gate driver stage. |
| **Power Switching** | KEC `KIA7N65H` | TO-220F | 650V, 7A N-channel Power MOSFET in a fully electrically isolated package. Delivers low $R_{DS(on)}$ ($1.4\ \Omega$), eliminating acoustic motor hum. |
| **Current Sensing** | `ZHT103` / `HWCT-5A` | Through-Hole | 5A rated AC current transformer (5mA secondary ratio). Paired with Microchip `MCP6001` op-amp to report real-time fan power consumption and detect stalled blades. |
| **Over-Temperature** | `10kΩ NTC` Thermistor | 0805 / Radial | Monitors MOSFET junction and enclosure temperature for automated thermal throttling and fire prevention. |
| **Surge Clamping** | `10D471K` Varistor | Radial Disc (10mm) | Metal Oxide Varistor (MOV) clamping mains voltage spikes up to 470V / 2.5 kA. |
| **Overcurrent Fuse** | `2010T1A250V` Micro Fuse | 2010 / Radial | 1A, 250V AC slow-blow time-lag fuse preventing catastrophic damage during hard short circuits. |
| **RS-422 Transceiver** | Maxim `MAX491E` | SOP-14 | Differential full-duplex transceiver protected by `SM712` asymmetrical TVS diodes. |
| **Input Multiplexer** | NXP/TI `74HC165D` | SOP-16 | Parallel-in serial-out shift register sampling the 8-bit DIP switch for hardware node addressing. |

---

## Operating Principles & Control Logic

### 1. Phase-Angle Speed Modulation
- Upon receiving a zero-crossing event from the `H11AA1`, an MCU hardware timer is reset.
- The timer counts down the firing delay $\tau$ corresponding to the desired speed percentage (0–100%).
- At timer expiration, a high-current trigger pulse is sent via the `HCPL-3120` to switch the `KIA7N65H` MOSFET.

### 2. Stall Threshold Protection
Induction ceiling fans exhibit non-linear low-speed torque characteristics. If driven below their aerodynamic stall threshold:
- The motor generates excessive heat without rotating.
- Back-EMF drops to zero, drawing destructive stall currents.
- The Fan Node firmware enforces a **calibrated minimum speed floor** (typically 15–20%). Any command requesting a speed below this threshold is automatically clamped to the minimum floor or switched completely `OFF`.

### 3. Soft Acceleration & Inrush Limiting
When transitioning from stop (`0%`) to a higher speed setting (e.g. `80%`), the firmware applies a smooth acceleration ramp over 1.5–2.5 seconds. This prevents inductive kickback, prolongs bearing life, and eliminates current spikes on the residential electrical circuit.

### 4. Telemetry Reporting
The Fan Controller reports telemetry directly to the central controller (Home Assistant `0xFFFF`):
- **Current & Power (`0x05` / `0x06`)**: Active wattage and RMS current computed from the current transformer.
- **Board Temperature (`0x0A`)**: Heatsink thermal telemetry from the 10kΩ NTC thermistor.
- **RPM / Fan State (`0x09`)**: Derived speed confirmation.

---

## Supported Protocol Commands

In accordance with the [HomeBus Protocol Specification](../../Firmware/README.md):

| Command Code | Command Name | Payload | Behavior |
| :---: | :--- | :--- | :--- |
| `0x01` | `SET_STATE` | `0x00` (OFF) / `0x01` (ON) | Switches fan completely off or returns to last active speed setting. |
| `0x02` | `TOGGLE` | None | Inverts current operating state. |
| `0x03` | `SET_PERCENT` | `0` to `100` (1 byte) | Sets precise speed percentage (clamped to stall threshold). |
| `0x04` | `INCREASE_PERCENT`| Delta `+1%` to `+100%` | Increments current speed by requested step (e.g. `+15%`). |
| `0x05` | `DECREASE_PERCENT`| Delta `-1%` to `-100%` | Decrements current speed by requested step (e.g. `-15%`). |
| `0x09` | `IDENTIFY` | Duration (seconds) | Flashes the onboard WS2812B NeoPixel in rapid pattern for physical locating. |

---

## Navigation

- [← Hardware Subsystem Overview](../README.md)
- [↑ HomeBus Root Documentation](../../README.md)
- [Base Node Documentation →](../base_node/README.md)
- [Gateway Node Documentation →](../Gateway_node/README.md)
- [Expander Daughterboards Documentation →](../Expanders/README.md)
- [Firmware & Protocol Specification →](../../Firmware/README.md)
