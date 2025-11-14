# Beacon-System-Sample

# BLE Distance Detection Exhibition System for Video/Audio Playback (No Wi-Fi)

---

# Overview

A system that **automatically plays video and audio when an object with ESP32 approaches an exhibition PC** without using Wi-Fi:

-   PC side: Automatically plays video
-   Object (ESP32) side: Automatically plays audio through wired speakers

The PC monitors RSSI (radio signal strength) via BLE and sends a GATT write (`PLAY`) to the target ESP32 when the threshold is exceeded. The ESP32 plays audio from microSD upon receipt. Audio output uses **wired speakers** (DFPlayer, or I2S → amplifier → speaker).

---

# Table of Contents

1. System Architecture (Simplified)
2. Recommended Equipment (per object)
3. Wiring Diagram
4. ESP32 (Arduino) Side — Implementation (BLE GATT Server + DFPlayer Control)
5. PC Side (Python) — Implementation (BLE Scan + RSSI Threshold → PLAY Send + Video Playback)
6. Calibration and Operational Notes
7. Deployment Checklist

---

# 1. System Architecture (Simplified)

```
[Object held by user]
  ├─ ESP32 (BLE Peripheral / GATT Server)
  ├─ DFPlayer Mini or I2S DAC + microSD (audio storage)
  ├─ Small amplifier (if needed) → wired speaker
  └─ Battery (mobile battery, etc.)

[Each exhibition PC]
  ├─ USB BLE dongle (or built-in BLE)
  ├─ Python (bleak) script: continuous scanning, RSSI judgment, GATT connection → PLAY write
  └─ Video playback software (VLC, etc.)
```

---

# 2. Recommended Equipment (per object)

-   ESP32-DevKitC (or M5Stack series) ×1
-   DFPlayer Mini (MP3 player module) ×1
    → Insert MP3 files into microSD for playback (simple implementation)
    ※ Alternative: ESP32 + I2S DAC (ES8388 / MAX98357A) + SD card library (slightly more complex implementation)
-   microSD card (audio files)
-   Small amplifier (PAM8403, etc.) ※ If DFPlayer doesn't have built-in amplifier
-   Speaker (8Ω 1–3W)
-   Battery: Mobile battery or Li-ion (several hours with ESP32 + DFPlayer, capacity design required)
-   USB BLE dongle (PC side. Tested on Windows and Linux)
-   PC: Windows / Linux / macOS (Python environment, VLC, etc.)

---

# 3. Wiring Diagram (Object Side)

-   ESP32 UART1 (TX/RX) → DFPlayer RX/TX
    (Example: ESP32 TX1(17) → DFPlayer RX, ESP32 RX1(16) ← DFPlayer TX)
-   DFPlayer VCC → 5V (regulated power supply / battery)
-   DFPlayer GND → ESP32 GND → Battery GND
-   DFPlayer SPK+/SPK- → Small amplifier input → Speaker (or DFPlayer output to amplifier)
-   DFPlayer BUSY (playback pin) → ESP32 GPIO (playback status flag)
-   (Optional) Connect a button for playback trigger to ESP32 GPIO

---

# 4. ESP32 (Arduino) Side — Code (BLE GATT Server + DFPlayer Control)

**Required Libraries**

-   ESP32 board definition (Arduino IDE / PlatformIO)
-   DFRobotDFPlayerMini library

```cpp
// ESP32 + DFPlayer Mini sample
// Overview: Sets up BLE GATT server, plays DFPlayer track 1 when "PLAY" is written to characteristic.
// Note: Adjust pin numbers according to your development board and wiring.

#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEServer.h>
#include <HardwareSerial.h>
#include <DFRobotDFPlayerMini.h>

#define SERVICE_UUID        "0000feed-0000-1000-8000-00805f9b34fb"
#define CHAR_CMD_UUID       "0000beef-0000-1000-8000-00805f9b34fb"

// Serial for DFPlayer (using UART1 as example)
HardwareSerial SerialDF(1); // UART1: TX=17, RX=16 (example)
DFRobotDFPlayerMini dfplayer;

BLECharacteristic *pCmdChar;
volatile bool isPlaying = false;
unsigned long lastPlayedAt = 0;
const unsigned long COOLDOWN_MS = 15000; // Cooldown to prevent re-triggering (ms)

// Pin number for BUSY pin to get playback status (DFPlayer BUSY → ESP32 GPIO)
const int DFPLAYER_BUSY_PIN = 4;

class MyCallbacks: public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *pCharacteristic) {
    std::string val = pCharacteristic->getValue();
    if(val.size() > 0) {
      String cmd = String((char*)val.c_str());
      if(cmd == "PLAY") {
        unsigned long now = millis();
        if(isPlaying) return;                 // Ignore if playing
        if(now - lastPlayedAt < COOLDOWN_MS) return; // Ignore during cooldown
        lastPlayedAt = now;
        // Play track 1 on DFPlayer (001.mp3 on microSD)
        dfplayer.play(1);
        isPlaying = true;
      }
    }
  }
};

void setupDFPlayer() {
  SerialDF.begin(9600, SERIAL_8N1, 16, 17); // RX=16, TX=17 example
  delay(200);
  if (!dfplayer.begin(SerialDF)) {
    Serial.println("DFPlayer init failed!");
    while(true) { delay(1000); } // Stop on init failure (retry design recommended for production)
  }
  dfplayer.volume(22); // 0-30
  pinMode(DFPLAYER_BUSY_PIN, INPUT_PULLUP); // If BUSY pin is available
}

void setup() {
  Serial.begin(115200);
  setupDFPlayer();

  // BLE initialization
  BLEDevice::init("OBJ_001"); // Use unique name for each object
  BLEServer *pServer = BLEDevice::createServer();
  BLEService *pService = pServer->createService(SERVICE_UUID);
  pCmdChar = pService->createCharacteristic(CHAR_CMD_UUID,
                                            BLECharacteristic::PROPERTY_WRITE);
  pCmdChar->setCallbacks(new MyCallbacks()));
  pService->start();
  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->start();
  Serial.println("BLE advertising started.");
}

void loop() {
  // Detect playback end via BUSY pin and reset flag
  static int lastBusyState = HIGH;
  int busy = digitalRead(DFPLAYER_BUSY_PIN);
  if (busy == LOW && lastBusyState == HIGH) {
    // Playback started (BUSY: LOW indicates playing on many models, verify)
    Serial.println("DFPlayer BUSY: playing");
  } else if (busy == HIGH && lastBusyState == LOW) {
    // Playback ended
    Serial.println("DFPlayer BUSY: stopped");
    isPlaying = false;
  }
  lastBusyState = busy;

  delay(50);
}
```

**Notes**

-   DFPlayer's BUSY pin can be used to distinguish playing/stopped states, recommended for reliable playback state management.
-   DFPlayer manages tracks by track number (001.mp3, etc.), which is convenient.
-   Add power decoupling (capacitors) and GND loop countermeasures to reduce noise.

---

# 5. PC Side (Python) — Code (bleak + VLC)

**Prerequisites**

-   Python 3.8+
-   `pip install bleak python-vlc`

```python
# pc_trigger.py
# Overview: Finds devices starting with OBJ_ via BLE scan, connects when RSSI threshold is exceeded,
#           writes 'PLAY' to GATT characteristic, and plays video with VLC.
# Note: RSSI threshold requires on-site calibration.

import asyncio
from bleak import BleakScanner, BleakClient
import subprocess
import time

# Configuration (adjust on-site)
TARGET_NAME_PREFIX = "OBJ_"        # Prefix of ESP32 advertise name
CMD_CHAR_UUID = "0000beef-0000-1000-8000-00805f9b34fb"
RSSI_THRESHOLD = -60               # Example: -60 (close). Adjust value (larger value = closer)
DEBOUNCE_SEC = 12                  # Prevent re-triggering for same device (seconds)
VIDEO_PATH = r"C:\videos\demo.mp4" # Path to video file (Windows example)
VLC_CMD = ["vlc", "--play-and-exit", VIDEO_PATH]  # VLC command (adjust for environment)

# Execution record (address → last trigger time)
last_triggered = {}

async def trigger_device(address):
    try:
        async with BleakClient(address, timeout=5.0) as client:
            # Send 'PLAY' via write (ASCII)
            await client.write_gatt_char(CMD_CHAR_UUID, b'PLAY')
            # Start video playback (non-blocking)
            subprocess.Popen(VLC_CMD)
            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Triggered {address}")
    except Exception as e:
        print("Trigger failed:", e)

async def scan_loop():
    scanner = BleakScanner()
    while True:
        devices = await scanner.discover(timeout=3.0)
        now = time.time()
        for d in devices:
            # Name filter
            name = d.name or ""
            if not name.startswith(TARGET_NAME_PREFIX):
                continue
            # RSSI check
            rssi = d.rssi
            if rssi is None:
                continue
            # Debounce (prevent rapid consecutive triggers)
            addr = d.address
            last = last_triggered.get(addr, 0)
            if now - last < DEBOUNCE_SEC:
                continue
            if rssi >= RSSI_THRESHOLD:
                # Trigger asynchronously (connection process causes brief wait)
                last_triggered[addr] = now
                asyncio.create_task(trigger_device(addr))
        await asyncio.sleep(0.5)

if __name__ == "__main__":
    asyncio.run(scan_loop())
```

**Notes**

-   Adjust the sign and comparison of `RSSI_THRESHOLD` according to environment (the code above triggers when "RSSI is greater than or equal to threshold (larger value = closer)").
-   If the `vlc` command is not in PATH on Windows, specify the full path. Same for Linux/macOS.
-   Set `DEBOUNCE_SEC` longer (around 10–30 seconds) according to on-site requirements.

---

# 6. PC Side (C#) — Windows.Devices.Bluetooth + VLC

## Overview

Detects Arduino via BLE and starts video + audio playback when in close proximity.  
Automatically stops when moved away.

## Required Libraries

-   .NET 6 or higher
-   NuGet packages:
    -   `Windows.Devices.Bluetooth`
    -   `Windows.Devices.Enumeration`
    -   `Vlc.DotNet.Forms`

## Code Example

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using System.Windows.Forms;
using Windows.Devices.Bluetooth;
using Windows.Devices.Bluetooth.Advertisement;
using Windows.Devices.Bluetooth.GenericAttributeProfile;
using Windows.Storage.Streams;
using Vlc.DotNet.Forms;

namespace BLE_VideoSync
{
    public class Program : Form
    {
        private BluetoothLEAdvertisementWatcher watcher;
        private VlcControl vlc;
        private DateTime lastSeen = DateTime.MinValue;
        private DateTime lastTriggerTime = DateTime.MinValue;
        private bool isNear = false;
        private bool isPlaying = false;
        private BluetoothLEDevice currentDevice;
        private GattCharacteristic targetCharacteristic;
        private Timer checkTimer;

        // Match UUIDs with Arduino side
        private const string TargetServiceUuid = "0000fff0-0000-1000-8000-00805f9b34fb";
        private const string TargetCharacteristicUuid = "0000fff1-0000-1000-8000-00805f9b34fb";

        // Parameter settings
        private const int NearThreshold = -70; // RSSI threshold
        private const int LostTimeout = 4000;  // Time to consider as lost (ms)
        private const int DebounceSec = 5;     // Re-trigger prevention time (seconds)

        [STAThread]
        public static void Main()
        {
            Application.EnableVisualStyles();
            Application.Run(new Program());
        }

        public Program()
        {
            Text = "BLE Distance Detection Player";
            Width = 1280;
            Height = 720;

            // VLC initialization
            vlc = new VlcControl();
            vlc.Dock = DockStyle.Fill;
            vlc.VlcLibDirectory = new System.IO.DirectoryInfo(@"C:\\Program Files\\VideoLAN\\VLC");
            Controls.Add(vlc);

            // BLE scan settings
            watcher = new BluetoothLEAdvertisementWatcher
            {
                ScanningMode = BluetoothLEScanningMode.Active
            };
            watcher.Received += OnAdvertisementReceived;
            watcher.Start();

            // Departure monitoring timer
            checkTimer = new Timer();
            checkTimer.Interval = 1000;
            checkTimer.Tick += CheckTimer_Tick;
            checkTimer.Start();

            Console.WriteLine("BLE scanning started...");
        }

        private async void OnAdvertisementReceived(BluetoothLEAdvertisementWatcher sender, BluetoothLEAdvertisementReceivedEventArgs args)
        {
            var uuids = args.Advertisement.ServiceUuids.Select(u => u.ToString());
            if (!uuids.Contains(TargetServiceUuid, StringComparer.OrdinalIgnoreCase))
                return;

            lastSeen = DateTime.Now;

            if (args.RawSignalStrengthInDBm > NearThreshold)
            {
                if (!isNear)
                {
                    // DEBOUNCE processing: ignore if not enough time has passed since last trigger
                    if ((DateTime.Now - lastTriggerTime).TotalSeconds < DebounceSec)
                        return;

                    isNear = true;
                    lastTriggerTime = DateTime.Now;

                    Console.WriteLine($"Proximity detected: RSSI={args.RawSignalStrengthInDBm} dBm");
                    await ConnectAndTriggerAsync(args.BluetoothAddress);
                }
            }
        }

        private async Task ConnectAndTriggerAsync(ulong address)
        {
            try
            {
                currentDevice = await BluetoothLEDevice.FromBluetoothAddressAsync(address);
                if (currentDevice == null)
                {
                    Console.WriteLine("BLE connection failed");
                    return;
                }

                var servicesResult = await currentDevice.GetGattServicesAsync();
                var service = servicesResult.Services.FirstOrDefault(s => s.Uuid.ToString().Equals(TargetServiceUuid, StringComparison.OrdinalIgnoreCase));
                if (service == null)
                {
                    Console.WriteLine("Service not found");
                    return;
                }

                var charsResult = await service.GetCharacteristicsAsync();
                targetCharacteristic = charsResult.Characteristics.FirstOrDefault(c => c.Uuid.ToString().Equals(TargetCharacteristicUuid, StringComparison.OrdinalIgnoreCase));
                if (targetCharacteristic == null)
                {
                    Console.WriteLine("Characteristic not found");
                    return;
                }

                // Send playback command to Arduino
                await WriteCommandAsync("PLAY");

                // Play video on PC
                PlayVideo("C:\\Videos\\demo.mp4");
                isPlaying = true;
            }
            catch (Exception ex)
            {
                Console.WriteLine("Connection error: " + ex.Message);
            }
        }

        private async Task WriteCommandAsync(string command)
        {
            if (targetCharacteristic == null) return;

            var writer = new DataWriter();
            writer.WriteString(command);
            await targetCharacteristic.WriteValueAsync(writer.DetachBuffer());
            Console.WriteLine($"Command sent: {command}");
        }

        private async void CheckTimer_Tick(object sender, EventArgs e)
        {
            if (isNear && (DateTime.Now - lastSeen).TotalMilliseconds > LostTimeout)
            {
                Console.WriteLine("Device departure detected");
                isNear = false;

                await WriteCommandAsync("STOP");
                StopVideo();
                isPlaying = false;
            }
        }

        private void PlayVideo(string path)
        {
            if (!isPlaying)
            {
                vlc.Play(new Uri(path));
                Console.WriteLine("Video playback started");
            }
        }

        private void StopVideo()
        {
            if (isPlaying)
            {
                vlc.Stop();
                Console.WriteLine("Video stopped");
            }
        }

        protected override void OnFormClosing(FormClosingEventArgs e)
        {
            watcher.Stop();
            checkTimer.Stop();
            vlc.Dispose();
            base.OnFormClosing(e);
        }
    }
}
```

---

# 7. Calibration and Operational Notes

-   **RSSI is environment-dependent**: RSSI varies significantly with metal, walls, human body, and angle, so always measure on-site and determine threshold.

    -   Recommended method: Place object at expected position (playback point), take 30–50 samples of RSSI obtained by PC at that position, and set average as threshold (also add hysteresis).

-   **Hysteresis**: Separate entry and exit thresholds to prevent chattering (example: entry -60, exit -70).
-   **First-come-first-served / Conflict**: When an object enters the boundary area of multiple PCs, the **PC that first establishes GATT connection and writes PLAY** triggers. This is the simplest approach. If needed, logically separate by zone assignment.
-   **Duplicate prevention during playback**: ESP32 side should ignore re-triggers during playback (determined by BUSY pin).
-   **Power management**: Monitor battery level and create backup charging schedule (important for exhibition operation).
-   **Security**: BLE without encryption may be triggered by unauthorized devices. If needed, introduce simple authentication (UID check) on connection.
-   **Logging**: PC side should log trigger time, device address, and RSSI for easier troubleshooting.

---

# 8. Deployment Checklist

-   [ ] Each object has unique BLE name (e.g., OBJ_001, OBJ_002) set
-   [ ] Playback files (001.mp3, etc.) are placed on microSD (format check)
-   [ ] DFPlayer BUSY pin is connected to ESP32 and can get playback status
-   [ ] PC BLE dongle can scan stably (driver/permissions)
-   [ ] RSSI calibration performed on-site and threshold determined (multiple samples)
-   [ ] DEBOUNCE/COOLDOWN values adjusted for actual use (e.g., 10–30 seconds)
-   [ ] Volume adjustment and noise countermeasures (power decoupling) implemented
-   [ ] Conflict rules for multiple PCs (first-come/zone assignment) decided and communicated to operations staff

---

# Appendix: Common Issues and Solutions

-   **Trigger flickers frequently (chattering)**
    → Introduce RSSI moving average + hysteresis + debounce (time judgment).
-   **Audio distortion / noise**
    → Add power decoupling (capacitors). Properly ground GND. Check amplifier ground connections.
-   **Playback conflicts occur when accessed simultaneously from multiple PCs**
    → ESP32 should "ignore writes during playback" or PC side should perform zone separation.
-   **Battery drains quickly**
    → Measure power consumption and increase battery capacity. Consider intermittent beacon output on ESP32 for power saving.
-   **Does USB BLE dongle need to be directly connected to PC? Can it work with power supply only?**
    → **It must be directly connected to PC**. It will not work with power supply only. USB BLE dongles communicate with PC's BLE stack via USB, so they must be connected to PC and provide BLE functionality via drivers. PC-side software (Python's bleak or C#'s Windows.Devices.Bluetooth) controls the dongle via PC's BLE stack. As an alternative, if the PC has a built-in BLE adapter, it can be used (many laptops and some desktop PCs have built-in BLE).

