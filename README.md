# ECG_Firmware — ESP32 ADC Bridge

> **Architecture (v2):** This firmware runs on the ESP32 as a **dumb USB ADC bridge only**.  
> No WiFi. No HTTP. No cloud logic. All AI runs on the Raspberry Pi 4B.

---

## What This Firmware Does

| Feature | Details |
|---------|---------|
| Sampling | 250 Hz via hardware timer ISR (4000 µs interval) |
| ADC | GPIO34, 12-bit (0–4095), AD8232 analog output |
| Smoothing | 3-sample moving average (reduces quantization noise) |
| Lead-off | GPIO32 (LO+), GPIO33 (LO-) — detects disconnected electrodes |
| Buzzer | GPIO25 — activated by RPi sending `BUZZ_ON` over USB Serial |
| Output | `<millis>,<ecg_value>,<lead_off>\n` at 115200 baud |

---

## Hardware Wiring

```
AD8232 OUTPUT  →  ESP32 GPIO34   (ADC input)
AD8232 LO+     →  ESP32 GPIO32   (Lead-off detect)
AD8232 LO-     →  ESP32 GPIO33   (Lead-off detect)
Buzzer (+)     →  ESP32 GPIO25   (Alert output)
AD8232 VCC     →  ESP32 3.3V
AD8232 GND     →  ESP32 GND
ESP32 USB      →  Raspberry Pi 4B USB port
```

---

## How to Flash (Arduino IDE)

### 1. Install Board Support
- Open Arduino IDE → **File → Preferences**
- Add this URL to "Additional boards manager URLs":
  ```
  https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
  ```
- Go to **Tools → Board → Boards Manager** → search `esp32` → install **esp32 by Espressif Systems** (v3.x)

### 2. Select Board & Port
- **Tools → Board** → `ESP32 Dev Module` (or your specific ESP32 variant)
- **Tools → Port** → select the COM port for your ESP32 (e.g., `COM3` on Windows)

### 3. Flash
- Open `ECG_Firmware.ino` in Arduino IDE
- Click **Upload** (→ arrow button)
- Wait for "Done uploading"

---

## Verify Serial Output

After flashing:
1. Open **Tools → Serial Monitor**
2. Set baud rate to **115200**
3. You should see CSV lines immediately:
   ```
   500,2048,0
   504,2051,0
   508,2045,0
   ```
   Format: `<millis>,<ecg_value>,<lead_off>`

**If `lead_off = 1`** → electrodes are disconnected. Attach the ECG leads to see real signal.

---

## Verify on Raspberry Pi (USB Connection)

After connecting ESP32 to RPi via USB cable:

```bash
# Find the serial port
ls /dev/tty*
# Should show /dev/ttyUSB0 or /dev/ttyACM0

# Read raw output (Ctrl+C to stop)
cat /dev/ttyUSB0
# or
python3 -c "
import serial
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
for _ in range(20):
    print(s.readline().decode().strip())
s.close()
"
```

Expected output on RPi:
```
1234,2048,0
1238,2050,0
1242,2047,0
```

---

## Test Buzzer Command

Send `BUZZ_ON` from RPi to ESP32 over Serial (this is what `realtime_inference.py` does automatically):

```python
import serial, time
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
s.write(b'BUZZ_ON\n')
time.sleep(2)
s.write(b'BUZZ_OFF\n')
s.close()
```

The buzzer on GPIO25 should activate for 2 seconds.

---

## Serial Protocol Reference

| Direction | Command / Data | Description |
|-----------|---------------|-------------|
| ESP32 → RPi | `<millis>,<ecg_val>,<lead_off>\n` | 250 samples/sec |
| RPi → ESP32 | `BUZZ_ON\n` | Activate alert buzzer |
| RPi → ESP32 | `BUZZ_OFF\n` | Deactivate buzzer |

---

## Common Issues

| Problem | Fix |
|---------|-----|
| No serial port in Arduino IDE | Install CP210x or CH340 USB driver for your ESP32 board |
| `analogSetAttenuation` error | Ensure ESP32 Arduino core **v3.x** is installed (not v2.x) |
| `timerBegin` / `timerAlarm` error | Same — v3.x API. Do not use old `timerBegin(0, 80, true)` syntax |
| All values = 0 or 4095 | Check ADC wiring. GPIO34 must have AD8232 OUTPUT connected |
| `lead_off` always = 1 | Electrodes not connected, or LO+/LO- wired incorrectly |
| Port permission denied on RPi | Run: `sudo usermod -aG dialout $USER` then **reboot** |

---

## Why No WiFi?

This firmware deliberately omits WiFi for these reasons (v2 architecture decision):

- **Reliability**: WiFi reconnects cause sample gaps; USB Serial is deterministic
- **Latency**: USB Serial has ~0ms overhead vs WiFi TCP overhead
- **Simplicity**: No WiFi credentials, no HTTP client, no JSON serialization on the ESP32
- **Power**: No WiFi radio = lower power consumption and heat
- **Security**: No network attack surface on the sensor node

The Raspberry Pi 4B handles all network communication (WiFi → MongoDB Atlas → Render cloud API).
