# can_wireless_sniffer

Wireless CAN sniffer based on **two ESP32-S3 boards**.

We already have a **wired CAN-USB adapter**:  
https://docs.raccoonlab.co/guide/programmer_sniffer/sniffer.html  

The goal of this project:  
**Do the same job (sniff CAN traffic to a PC), but wirelessly.**  
This should help when it is hard to access the CAN wiring (for example, on a flying drone).

---

## 1. Overview

There are two devices:

- **Transmitter** – connects to the **CAN bus** of the drone.
- **Receiver** – connects to the **PC** via **USB**.

The Transmitter reads CAN frames and sends them over **ESP-NOW**.  
The Receiver gets these ESP-NOW packets and sends the CAN frames to the PC over **USB** in **SLCAN** format.  
On the PC we read these frames using **python-can** (Windows is OK).

---

## 2. User story (how it should work)

1. User plugs the **Receiver** to a PC (it should appear as a virtual COM port).
2. User runs a simple Python script using **python-can** which reads CAN frames from this COM port.
3. User powers the **Transmitter**, already connected to the CAN bus.
4. After a short time, the user sees CAN traffic on the PC.

Important:  
For the user this should be as **plug & play** as possible:

- No Wi-Fi SSID or password configuration.
- No switching Wi-Fi networks on the laptop.
- No extra “open this app, then type this IP” flow.

Plug Receiver → run program → power Transmitter → see CAN traffic.

---

## 3. Architecture

### 3.1 Transmitter

**Hardware:**

- ESP32-S3 dev board
- CAN transceiver (TJA… or similar)
- JST-4 connector to an existing CAN node

**Logic:**

- Configure CAN bus at **1 Mbit/s**.
- Read CAN frames from the bus.
- Pack frames into a custom binary format.
- Send packets to the Receiver using **ESP-NOW**.

### 3.2 Receiver

**Hardware:**

- ESP32-S3 dev board
- USB connection to PC (as **USB-CDC** / virtual COM port)

**Logic:**

- Receive packets over **ESP-NOW**.
- Decode packets into CAN frames.
- Encode each CAN frame into **SLCAN** ASCII line.
- Send SLCAN lines to the PC over the USB serial port.
- On the PC side, use **python-can** to read these frames as from a normal SLCAN adapter.

---

## 4. Functional requirements

### 4.1 CAN interface

- Bitrate: **1 Mbit/s** (this is enough for the first version).
- Support **standard CAN IDs** (11-bit).
- Extended IDs (29-bit) are optional and can be done later.

### 4.2 Internal frame format

On the Transmitter and Receiver we use a binary `struct` (or similar) for one CAN frame:

- `can_id` – 32-bit integer (same style as Linux CAN: ID + flags)
- `dlc` – data length code (0–8)
- `data[8]` – up to 8 bytes of payload
- `timestamp` – 32-bit integer, for example milliseconds from device startup

We can pack **several CAN frames into one ESP-NOW packet** to reduce overhead.

### 4.3 ESP-NOW

- Transmitter knows the **MAC address** of the Receiver (hard-coded for now).
- Both devices use a fixed **Wi-Fi channel** (hard-coded).
- Transmitter sends packets as soon as there is data (or on a short timer).

### 4.4 USB / SLCAN

- Receiver uses **USB-CDC** → appears as a COM port on Windows.
- Each CAN frame is converted to a **SLCAN** ASCII string ending with `\r`.
- On the PC, **python-can** opens this COM port as a SLCAN interface and reads frames.

### 4.5 Direction of traffic

- First version: **only CAN → PC**.
- Sending frames **from PC to CAN** is not required now.

---

## 5. Non-functional requirements

Keep it simple, but try to reach this:

1. **Latency**  
   Extra delay from CAN bus to PC should be roughly **+1–2 ms** compared to the wired adapter (we will measure).

2. **Throughput**  
   System should work with CAN bus load **50–80%** at 1 Mbit/s without big frame loss.

3. **Stateless behavior**  
   If Transmitter or Receiver is reset or powered off:
   - The other device should continue to work.
   - When the device comes back, the link should recover automatically.
   - No need to press reset on both sides.

---

## 6. Git workflow

- One **shared repository** for the whole project.
- Suggested structure:
  - `main` – stable branch (working prototype, tested).
  - `tx/*` – branches for Transmitter work.
  - `rx/*` – branches for Receiver work.
- Use pull requests into `main` once some part is tested.

---

## 7. Workstreams

We split the work into two **streams** that can be done in parallel.

### 7.1 Common setup (for both people)

1. **GitHub & repo**
   - Create or use your GitHub account.
   - Accept access to this repository.
   - Clone the repo and make a first test commit.

2. **Toolchain**
   - Install toolchain for ESP32-S3:
     - Arduino **or**
     - PlatformIO **or**
     - ESP-IDF  
     (Choose one. Write in the README which one you use and how to build.)

3. **Hello world**
   - Build and flash a simple example:
     - Blink an LED
     - Print something to serial
   - Make sure ESP32-S3 and USB connection work.

---

### 7.2 Workstream: Transmitter

Goal: **CAN → ESP-NOW**

1. **CAN basic**
   - Configure CAN for 1 Mbit/s.
   - Connect CAN transceiver and a CAN test node.
   - Read CAN frames and print them to UART in a human-readable form.

2. **Internal frame struct**
   - Implement `struct` for CAN frame (see 4.2).
   - Implement function like `bool read_can_frame(can_frame_t* frame)`.

3. **ESP-NOW send (test)**
   - Set up ESP-NOW in **sender** mode.
   - Hard-code Receiver MAC.
   - Send a simple test packet (for example, `"Hello"` or fake frames).

4. **CAN → ESP-NOW**
   - Replace test data with real CAN frames.
   - Optionally pack several frames into one ESP-NOW packet.
   - Decide when to send:
     - when buffer is full, or
     - every ~1 ms with whatever is in buffer.

---

### 7.3 Workstream: Receiver

Goal: **ESP-NOW → USB/SLCAN**

1. **ESP-NOW receive (test)**
   - Set up ESP-NOW in **receiver** mode.
   - Receive test packets from Transmitter.
   - Print packet content to UART.

2. **Parse internal format**
   - Decode binary packets into CAN frames.
   - (Optional) Add `seq_id` field to packets to count lost packets.
   - Print some statistics: number of frames, number of packets, lost packets.

3. **USB-CDC**
   - Enable USB-CDC on ESP32-S3.
   - Check that PC sees it as a COM port.
   - Send simple test lines over USB serial (e.g. `"hello\r\n"`).

4. **SLCAN encoder**
   - Implement function that converts `can_frame_t` → SLCAN ASCII string + `\r`.
   - For each received CAN frame, send SLCAN string to USB-CDC.

5. **python-can demo**
   - Write a **simple Python script** that:
     - uses `python-can`,
     - opens the COM port as SLCAN interface,
     - prints first N frames to console.
   - Test this script on Windows.

---

## 8. Testing

Three simple tests for the prototype:

### 8.1 Latency test

- Add timestamps to frames or logs.
- Measure time from CAN frame reception on Transmitter to frame arrival on PC (for example, by logging timestamps on both sides).
- Estimate:
  - average latency,
  - maximum latency.

### 8.2 Stress test

- Generate CAN traffic with **50–80% bus load** at 1 Mbit/s.
- Run the system for some time.
- Check:
  - number of frames sent vs. received,
  - approximate packet loss (preferably using a counter or `seq_id`).

### 8.3 Reconnect / reset test

- With active CAN traffic:
  - Reset or power-cycle the **Transmitter** while Receiver stays on.
  - Reset or power-cycle the **Receiver** while Transmitter stays on.
- Expected behavior:
  - After device is back, frames start flowing again.
  - No need to manually reset the other side or restart the PC script.

---

## 9. Plan B (if ESP32-S3 / ESP-NOW fails)

If there are big problems with ESP32-S3 or ESP-NOW:

- Use older Wi-Fi bridge hardware (if available).
- Implement the same idea:
  - Device 1: CAN → Wi-Fi (UDP or similar),
  - Device 2: Wi-Fi → USB/SLCAN → PC.
- Repeat the basic tests:
  - does traffic reach PC,
  - what is latency,
  - how stable it is.

The main goal is to **prove the concept** of a wireless CAN sniffer, even if the first version is not perfect.
