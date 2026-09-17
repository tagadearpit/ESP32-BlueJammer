# ESP32-BlueJammer Battery & Power Wiring Guide

This guide provides a detailed, safe wiring reference for adding a rechargeable lithium-ion battery (3.7V nominal, 4.2V peak) to the **ESP32-BlueJammer** using a **TP4056 charging module** and a **mini slide switch**.

---

## 1. Safety Warning & Voltage Regulation Analysis

### The 4.2V Overvoltage Issue on 3V3 Pin
A standard 1-cell (1S) Li-Ion or Li-Po battery outputs **3.7V nominal** and charges up to **4.2V (±0.05V)** under full charge.

According to the official **Espressif ESP32 Datasheet (Section 4.1 "Absolute Maximum Ratings")**:
- **Power supply voltage ($V_{DD}$):** $-0.3\text{ V}$ to $+3.6\text{ V}$
- **Operating voltage range ($V_{DD}$):** $3.0\text{ V}$ to $3.6\text{ V}$ (Typical: $3.3\text{ V}$)

Connecting the battery output directly to the **3V3 pin** of the ESP32 development board connects to the 3.3V rail **downstream of the on-board low-dropout (LDO) regulator**. At 4.2V:
1. It **exceeds the absolute maximum rating of 3.6V** by 600 mV.
2. It exposes the ESP32 microcontroller, flash memory, and any attached 3.3V peripherals (like the nRF24L01+ modules and OLED display) to destructive overvoltage.
3. Over time or immediately, this causes overheating, brownout errors, degraded radio performance, or permanent failure of the ESP32.

### The Solution: Route via the VIN / 5V Pin
ESP32 development boards (such as ESP32-WROOM-32U DevKitC, NodeMCU-32S, ESP32-DevKit V1) feature an integrated on-board linear LDO regulator (typically an **AMS1117-3.3**, **ME6211**, or **NCP1117**):
- The **VIN / 5V** pin feeds directly into the input ($V_{IN}$) of this onboard regulator.
- The regulator takes an input voltage of 3.7V–5.5V and outputs a clean, stable **3.3V** to the ESP32 chip and 3.3V power rails.
- Operating the LDO with a 3.7V–4.2V input requires very little voltage drop (only 0.4V–0.9V drop), generating minimal heat while providing complete overvoltage protection for the MCU and radio modules.

> **Always connect the battery switch output to the ESP32's `VIN` (or `5V`) pin, NEVER directly to `3V3`!**

---

## 2. Complete Wiring Diagram

```
 +------------------------+
 | 3.7V Li-Ion / Li-Po    |
 | Battery (1S)           |
 +-----------+------------+
      (+)    |    (-)
      |      |
      v      v
 +----+------+------------+
 | JST-PH 2.0 Connector   |
 +----+------+------------+
      |      |
      |      +-------------------------------------------+
      |                                                  |
      v (+)                                              v (-)
   [BAT +]                                            [BAT -]
 +---------------------------------------------------------------+
 |                  TP4056 Charging Module                       |
 |                                                               |
 | [Type-C / Micro-USB Port]                                     |
 | (For battery charging only)                                   |
 |                                                               |
 |   [OUT +]                                          [OUT -]    |
 +------+------------------------------------------------+-------+
        |                                                |
        v                                                |
   +----+------------------+                             |
   | Mini Slide Switch     |                             |
   | (Pin 1: In / Center)  |                             |
   | (Pin 2: Out / Switched|                             |
   +----+------------------+                             |
        |                                                |
        | Switched Battery Power                         |
        |                                                |
        v                                                v
   +----+------------------------------------------------+-------+
   |                        ESP32 Board                          |
   |                                                             |
   |   [VIN / 5V]                                     [GND]      |
   |   (Input to onboard 3.3V regulator)                         |
   +----+------------------------------------------------+-------+
        |                                                |
        v Regulated 3.3V Rail                            |
   +----+------------------------------------------------+-------+
   | nRF24L01+ Modules & OLED Display                            |
   |                                                             |
   |   [VCC] (3.3V)                                   [GND]      |
   +-------------------------------------------------------------+
```

---

## 3. Pin-to-Pin Connection Table

| Component Source | Terminal / Pin | Target Component | Target Pin | Purpose & Notes |
| :--- | :--- | :--- | :--- | :--- |
| **3.7V Li-Ion Battery** | Positive $(+)$ Wire | JST-PH 2.0 Male | $(+)$ Pin (Red) | Battery supply connection |
| **3.7V Li-Ion Battery** | Negative $(-)$ Wire | JST-PH 2.0 Male | $(-)$ Pin (Black) | Battery ground connection |
| **JST-PH 2.0 Female** | Positive $(+)$ Lead | TP4056 Module | **BAT +** | Battery charging input $(+)$ |
| **JST-PH 2.0 Female** | Negative $(-)$ Lead | TP4056 Module | **BAT -** | Battery charging input $(-)$ |
| **TP4056 Module** | **OUT +** | Mini Slide Switch | **Pin 1 (Input)** | Power feed from charger/protection circuit |
| **Mini Slide Switch** | **Pin 2 (Switched Out)** | ESP32 Dev Board | **VIN** or **5V** | **Feeds onboard regulator (Safe 3.3V step-down)** |
| **TP4056 Module** | **OUT -** | ESP32 Dev Board | **GND** | Common ground return |
| **ESP32 Dev Board** | **3V3** | nRF24L01+ (x2) & OLED | **VCC** | Regulated 3.3V output for peripherals |
| **ESP32 Dev Board** | **GND** | nRF24L01+ (x2) & OLED | **GND** | Common ground for peripherals |

---

## 4. Optional Status LED & Resistor Wiring

If you are using the status indicator LED:

```
 ESP32 GPIO27 ---->[ 4.7kΩ Resistor ]---->(+) [ 3mm Blue LED ] (-)----> ESP32 GND
```

- **Resistor:** 4.7kΩ connected in series with the positive (anode, longer lead) of the LED to limit current from GPIO27.
- **Cathode (shorter lead / flat edge):** Connected directly to ESP32 **GND**.

---

## 5. Critical USB Power & Charging Precautions

1. **Charging the Battery via TP4056:**
   - Plug a USB-C or Micro-USB cable into the **TP4056 charging port**.
   - Keep the **Mini Slide Switch in the OFF position** while charging so the battery charges efficiently without powering the ESP32 and transmitters.
   - Indicator LEDs on the TP4056:
     - **Red LED ON:** Battery is actively charging.
     - **Blue / Green LED ON:** Battery is fully charged (4.2V reached).

2. **Flashing or Programming the ESP32 via PC USB:**
   - **TURN THE SLIDE SWITCH OFF** before plugging a USB data cable into the ESP32 development board's native USB port.
   - Leaving the battery switch ON while plugging into PC USB can cause two active voltage sources (USB 5V and Battery 4.2V) to conflict on the board, potentially backfeeding current into the TP4056 or your computer's USB port.

---

## 6. Pre-Power Verification Checklist

Before turning on the switch for the first time:
- [ ] Verify battery polarity on the JST connector: Red is $(+)$, Black is $(-)$.
- [ ] Ensure the TP4056 `OUT +` goes to the switch, and the switch output connects to **VIN / 5V** on the ESP32 (verify it is **NOT** soldered to `3V3`).
- [ ] Verify with a digital multimeter that resistance between ESP32 `VIN` and `GND` is not shorted ($>1\text{ k}\Omega$).
- [ ] Turn on the slide switch and measure the voltage between ESP32 `3V3` and `GND`: it should read a stable **$3.28\text{ V} - 3.33\text{ V}$**.
- [ ] If the reading is anywhere above $3.6\text{ V}$, turn off immediately and verify that the switch output is connected to **VIN / 5V** rather than **3V3**.
