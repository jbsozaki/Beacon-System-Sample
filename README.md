# Beacon-System-Sample

# BLE 距離検知で映像・音声を再生する展示システム（Wi-Fi 無し）

---

# 概要

Wi-Fi を使わず、**ESP32 を持ったオブジェクトが展示 PC の近傍に入ったら**

-   PC 側：映像を自動再生
-   オブジェクト（ESP32）側：有線スピーカーで音声を自動再生

を実現するシステムです。PC は BLE で RSSI（電波強度）を監視し閾値を越えたら対象 ESP32 へ GATT 書き込み（`PLAY`）を送り、ESP32 は受信したら microSD 内の音声を再生します。音声出力は**有線スピーカー**（DFPlayer 等、もしくは I2S→ アンプ → スピーカー）です。

---

# 目次

1. 構成図（簡略）
2. 推奨機材（1 オブジェクト分）
3. 配線イメージ
4. ESP32（Arduino）側 — 実装（BLE GATT サーバ + DFPlayer 制御）
5. PC 側（Python） — 実装（BLE スキャン＋ RSSI 閾値 →PLAY 送信＋映像再生）
6. キャリブレーションと運用上の注意
7. デプロイ時チェックリスト

---

# 1. 構成図（簡略）

```
[ユーザーが持つオブジェクト]
  ├─ ESP32 (BLEペリフェラル / GATTサーバ)
  ├─ DFPlayer Mini or I2S DAC + microSD（音声保存）
  ├─ 小型アンプ（必要なら） → 有線スピーカー
  └─ バッテリ（モバイルバッテリ等）

[各設置PC]
  ├─ USB BLEドングル（または内蔵BLE）
  ├─ Python (bleak) スクリプト：常時スキャン、RSSI判定、GATT接続→PLAY書き込み
  └─ 映像再生ソフト（VLC等）
```

---

# 2. 推奨機材（1 オブジェクト分）

-   ESP32-DevKitC（または M5Stack 系） ×1
-   DFPlayer Mini（MP3 プレーヤーモジュール） ×1
    → microSD に MP3 を入れて再生（簡単実装）
    ※ 代替：ESP32 + I2S DAC（ES8388 / MAX98357A）＋ SD カードライブラリ（実装はやや複雑）
-   microSD カード（音声ファイル）
-   小型アンプ（PAM8403 等）※DFPlayer にアンプ内蔵でない場合
-   スピーカー（8Ω 1–3W 程度）
-   バッテリ：モバイルバッテリ or Li-ion（ESP32 + DFPlayer で数時間〜、容量設計要）
-   USB BLE ドングル（PC 側。Windows や Linux で動作確認済みのもの）
-   PC：Windows / Linux / macOS（Python 環境、VLC 等）

---

# 3. 配線イメージ（オブジェクト側）

-   ESP32 UART1 (TX/RX) → DFPlayer RX/TX
    （例：ESP32 TX1(17) → DFPlayer RX, ESP32 RX1(16) ← DFPlayer TX）
-   DFPlayer VCC → 5V（安定化電源／バッテリ）
-   DFPlayer GND → ESP32 GND → バッテリ GND
-   DFPlayer SPK+/SPK- → 小型アンプ入力 → スピーカー（もしくは DFPlayer の出力をアンプへ）
-   DFPlayer BUSY（再生中ピン） → ESP32 GPIO（再生中フラグ取得）
-   （オプション）再生トリガ用のボタンを ESP32 の GPIO に接続

---

# 4. ESP32（Arduino）側 — コード（BLE GATT サーバ + DFPlayer 制御）

**前提ライブラリ**

-   ESP32 ボード定義（Arduino IDE / PlatformIO）
-   DFRobotDFPlayerMini ライブラリ

```cpp
// ESP32 + DFPlayer Mini のサンプル
// 概要：BLE GATT サーバを立て、特性に "PLAY" が書き込まれたら DFPlayer のトラック1を再生。
// 注意：ピン番号は開発ボードや配線に合わせて調整してください。

#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEServer.h>
#include <HardwareSerial.h>
#include <DFRobotDFPlayerMini.h>

#define SERVICE_UUID        "0000feed-0000-1000-8000-00805f9b34fb"
#define CHAR_CMD_UUID       "0000beef-0000-1000-8000-00805f9b34fb"

// DFPlayer 用シリアル（UART1 を使用する例）
HardwareSerial SerialDF(1); // UART1: TX=17, RX=16 (例)
DFRobotDFPlayerMini dfplayer;

BLECharacteristic *pCmdChar;
volatile bool isPlaying = false;
unsigned long lastPlayedAt = 0;
const unsigned long COOLDOWN_MS = 15000; // 再トリガーを防ぐクールダウン（ms）

// BUSYピンで再生状態を取得する場合のピン番号（DFPlayerのBUSY→ESP32 GPIO）
const int DFPLAYER_BUSY_PIN = 4;

class MyCallbacks: public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *pCharacteristic) {
    std::string val = pCharacteristic->getValue();
    if(val.size() > 0) {
      String cmd = String((char*)val.c_str());
      if(cmd == "PLAY") {
        unsigned long now = millis();
        if(isPlaying) return;                 // 再生中は無視
        if(now - lastPlayedAt < COOLDOWN_MS) return; // クールダウン中は無視
        lastPlayedAt = now;
        // DFPlayer でトラック1を再生（microSDの 001.mp3）
        dfplayer.play(1);
        isPlaying = true;
      }
    }
  }
};

void setupDFPlayer() {
  SerialDF.begin(9600, SERIAL_8N1, 16, 17); // RX=16, TX=17 の例
  delay(200);
  if (!dfplayer.begin(SerialDF)) {
    Serial.println("DFPlayer init failed!");
    while(true) { delay(1000); } // 初期化失敗なら停止（実運用ではリトライ設計）
  }
  dfplayer.volume(22); // 0-30
  pinMode(DFPLAYER_BUSY_PIN, INPUT_PULLUP); // BUSYピンがある場合
}

void setup() {
  Serial.begin(115200);
  setupDFPlayer();

  // BLE初期化
  BLEDevice::init("OBJ_001"); // 各オブジェクトでユニークな名前にする
  BLEServer *pServer = BLEDevice::createServer();
  BLEService *pService = pServer->createService(SERVICE_UUID);
  pCmdChar = pService->createCharacteristic(CHAR_CMD_UUID,
                                            BLECharacteristic::PROPERTY_WRITE);
  pCmdChar->setCallbacks(new MyCallbacks());
  pService->start();
  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->start();
  Serial.println("BLE advertising started.");
}

void loop() {
  // BUSYピンで再生終了を検知してフラグを戻す
  static int lastBusyState = HIGH;
  int busy = digitalRead(DFPLAYER_BUSY_PIN);
  if (busy == LOW && lastBusyState == HIGH) {
    // 再生が始まった（BUSY: LOWで再生中の機種が多いので確認）
    Serial.println("DFPlayer BUSY: playing");
  } else if (busy == HIGH && lastBusyState == LOW) {
    // 再生が終わった
    Serial.println("DFPlayer BUSY: stopped");
    isPlaying = false;
  }
  lastBusyState = busy;

  delay(50);
}
```

**備考**

-   DFPlayer の BUSY ピンは再生中/停止の判別に使えるので、確実な再生状態管理に推奨します。
-   DFPlayer ではトラック番号（001.mp3 等）で管理するのが楽です。
-   電源デカップリング（コンデンサ）や GND ループ対策をして、ノイズを減らしてください。

---

# 5. PC 側（Python） — コード（bleak + VLC）

**前提**

-   Python 3.8+
-   `pip install bleak python-vlc`

```python
# pc_trigger.py
# 概要：BLEスキャンで OBJ_ で始まるデバイスを見つけ、RSSI閾値を超えたら接続して
#        GATT特性に 'PLAY' を書き込み、VLCで動画を再生する。
# 注意：RSSI閾値は現地でキャリブレーションが必要です。

import asyncio
from bleak import BleakScanner, BleakClient
import subprocess
import time

# 設定項目（現地で要調整）
TARGET_NAME_PREFIX = "OBJ_"        # ESP32のadvertise名のプレフィックス
CMD_CHAR_UUID = "0000beef-0000-1000-8000-00805f9b34fb"
RSSI_THRESHOLD = -60               # 例: -60 (近い)。値は要調整（大きい数値=より近い）
DEBOUNCE_SEC = 12                  # 同一デバイス再トリガー防止（秒）
VIDEO_PATH = r"C:\videos\demo.mp4" # 再生する動画のパス（Windows例）
VLC_CMD = ["vlc", "--play-and-exit", VIDEO_PATH]  # VLCコマンド（環境に合わせて）

# 実行記録（アドレス→最終トリガー時刻）
last_triggered = {}

async def trigger_device(address):
    try:
        async with BleakClient(address, timeout=5.0) as client:
            # 書き込みで 'PLAY' を送る（ASCII）
            await client.write_gatt_char(CMD_CHAR_UUID, b'PLAY')
            # 映像再生（非ブロッキング）
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
            # 名前フィルタ
            name = d.name or ""
            if not name.startswith(TARGET_NAME_PREFIX):
                continue
            # RSSIチェック
            rssi = d.rssi
            if rssi is None:
                continue
            # デバウンス（短時間に連続トリガーしない）
            addr = d.address
            last = last_triggered.get(addr, 0)
            if now - last < DEBOUNCE_SEC:
                continue
            if rssi >= RSSI_THRESHOLD:
                # トリガーを非同期で実施（接続処理で短時間待ちになるため）
                last_triggered[addr] = now
                asyncio.create_task(trigger_device(addr))
        await asyncio.sleep(0.5)

if __name__ == "__main__":
    asyncio.run(scan_loop())
```

**備考**

-   `RSSI_THRESHOLD` の符号と比較は環境により調整してください（上のコードは「RSSI が閾値以上（値が大きい＝近い）」でトリガー）。
-   Windows で`vlc`コマンドが PATH に無い場合はフルパス指定にしてください。Linux/macOS も同様。
-   `DEBOUNCE_SEC` は現場の要件に合わせて長めに（10〜30 秒程度）設定すると良いです。

---

# 6. PC 側（C#）— Windows.Devices.Bluetooth + VLC

## 概要

BLE で Arduino を検出し、近距離に入ると映像＋音声再生を開始。  
離れると自動停止します。

## 必要ライブラリ

-   .NET 6 以上
-   NuGet パッケージ：
    -   `Windows.Devices.Bluetooth`
    -   `Windows.Devices.Enumeration`
    -   `Vlc.DotNet.Forms`

## コード例

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

        // Arduino側と一致させるUUID
        private const string TargetServiceUuid = "0000fff0-0000-1000-8000-00805f9b34fb";
        private const string TargetCharacteristicUuid = "0000fff1-0000-1000-8000-00805f9b34fb";

        // パラメータ設定
        private const int NearThreshold = -70; // RSSI閾値
        private const int LostTimeout = 4000;  // 離脱とみなす時間（ms）
        private const int DebounceSec = 5;     // 再トリガー防止時間（秒）

        [STAThread]
        public static void Main()
        {
            Application.EnableVisualStyles();
            Application.Run(new Program());
        }

        public Program()
        {
            Text = "BLE距離判定プレーヤー";
            Width = 1280;
            Height = 720;

            // VLC初期化
            vlc = new VlcControl();
            vlc.Dock = DockStyle.Fill;
            vlc.VlcLibDirectory = new System.IO.DirectoryInfo(@"C:\\Program Files\\VideoLAN\\VLC");
            Controls.Add(vlc);

            // BLEスキャン設定
            watcher = new BluetoothLEAdvertisementWatcher
            {
                ScanningMode = BluetoothLEScanningMode.Active
            };
            watcher.Received += OnAdvertisementReceived;
            watcher.Start();

            // 離脱監視タイマー
            checkTimer = new Timer();
            checkTimer.Interval = 1000;
            checkTimer.Tick += CheckTimer_Tick;
            checkTimer.Start();

            Console.WriteLine("BLEスキャン開始...");
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
                    // DEBOUNCE処理：前回から一定時間経過していなければ無視
                    if ((DateTime.Now - lastTriggerTime).TotalSeconds < DebounceSec)
                        return;

                    isNear = true;
                    lastTriggerTime = DateTime.Now;

                    Console.WriteLine($"接近検知: RSSI={args.RawSignalStrengthInDBm} dBm");
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
                    Console.WriteLine("BLE接続失敗");
                    return;
                }

                var servicesResult = await currentDevice.GetGattServicesAsync();
                var service = servicesResult.Services.FirstOrDefault(s => s.Uuid.ToString().Equals(TargetServiceUuid, StringComparison.OrdinalIgnoreCase));
                if (service == null)
                {
                    Console.WriteLine("サービスが見つかりません");
                    return;
                }

                var charsResult = await service.GetCharacteristicsAsync();
                targetCharacteristic = charsResult.Characteristics.FirstOrDefault(c => c.Uuid.ToString().Equals(TargetCharacteristicUuid, StringComparison.OrdinalIgnoreCase));
                if (targetCharacteristic == null)
                {
                    Console.WriteLine("Characteristicが見つかりません");
                    return;
                }

                // Arduinoへ再生指示
                await WriteCommandAsync("PLAY");

                // PCで映像再生
                PlayVideo("C:\\Videos\\demo.mp4");
                isPlaying = true;
            }
            catch (Exception ex)
            {
                Console.WriteLine("接続エラー: " + ex.Message);
            }
        }

        private async Task WriteCommandAsync(string command)
        {
            if (targetCharacteristic == null) return;

            var writer = new DataWriter();
            writer.WriteString(command);
            await targetCharacteristic.WriteValueAsync(writer.DetachBuffer());
            Console.WriteLine($"コマンド送信: {command}");
        }

        private async void CheckTimer_Tick(object sender, EventArgs e)
        {
            if (isNear && (DateTime.Now - lastSeen).TotalMilliseconds > LostTimeout)
            {
                Console.WriteLine("デバイス離脱検出");
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
                Console.WriteLine("映像再生開始");
            }
        }

        private void StopVideo()
        {
            if (isPlaying)
            {
                vlc.Stop();
                Console.WriteLine("映像停止");
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

# 7. キャリブレーションと運用上の注意

-   **RSSI は環境依存**：金属、壁、人体、角度で大きく変動するため、現地で必ず測定して閾値を決めること。

    -   推奨法：オブジェクトを想定位置（再生ポイント）に置き、その位置で PC が取得する RSSI を 30 ～ 50 サンプル取り、平均を閾値設定（さらにヒステリシスを入れる）。

-   **ヒステリシス**：入る閾値と出る閾値を分けてチャタリングを防ぐ（例：入る -60、出る -70）。
-   **先着制/競合**：オブジェクトが複数 PC の境界領域に入った場合、**先に GATT 接続できて PLAY を書き込めた PC**がトリガーされる運用が簡単。必要ならゾーン割り当てで論理的に分離する。
-   **再生中の重複排除**：ESP32 側で再生中は再トリガーを無視（BUSY ピンで判別）すること。
-   **電源管理**：バッテリ残量監視や予備充電スケジュールを作る（展示運用では重要）。
-   **セキュリティ**：BLE は暗号化していない場合、野良端末の誤トリガーの可能性がある。必要なら接続時に簡易認証（UID チェック）を導入する。
-   **ログ**：PC 側はトリガー時刻・デバイスアドレス・RSSI をログに残すとトラブル解析が楽。

---

# 8. デプロイ時チェックリスト

-   [ ] 各オブジェクトにユニークな BLE 名（例：OBJ_001, OBJ_002）を設定しているか
-   [ ] microSD に再生用ファイル（001.mp3 等）を配置しているか（フォーマット確認）
-   [ ] DFPlayer の BUSY ピンを ESP32 に接続して再生状態を取得できるか
-   [ ] PC の BLE ドングルが安定してスキャンできるか（ドライバ/権限）
-   [ ] 現地で RSSI キャリブレーションを実施し閾値を決めたか（複数サンプル）
-   [ ] DEBOUNCE/COOLDOWN 値を実使用に合わせて調整したか（例：10〜30 秒）
-   [ ] 音量調整、ノイズ対策（電源デカップリング）を実施したか
-   [ ] 複数 PC での競合ルール（先着/ゾーン割り当て）を決め、運用担当に周知したか

---

# 付録：よくある問題と対策

-   **トリガーが頻繁にチラつく（チャタリング）**
    → RSSI の移動平均 + ヒステリシス + デバウンス（時間判定）を導入。
-   **音が割れる・ノイズがある**
    → 電源のデカップリング（コンデンサ）を入れる。GND を正しくまとめる。アンプのアース回りを確認。
-   **複数 PC から同時にアクセスされ再生競合が起きる**
    → ESP32 で「再生中は書き込みを無視」するか、PC 側でゾーン分けを行う。
-   **バッテリがすぐ切れる**
    → 消費電力を測り、バッテリ容量を増やす。ESP32 のビーコン出力を間欠にして省電力化も検討。
