# BLE Distance Detection Exhibition System - Sequence Diagrams

This document describes the operation flow of a BLE distance detection exhibition system for video/audio playback using sequence diagrams.

## Main Sequence: Playback Flow When Object Approaches

```mermaid
sequenceDiagram
    participant User as User<br/>(Holds Object)
    participant ESP32 as ESP32<br/>(BLE Peripheral)
    participant DFPlayer as DFPlayer Mini<br/>(Audio Playback)
    participant PC as Exhibition PC<br/>(BLE Scan)
    participant VLC as VLC<br/>(Video Playback)

    Note over ESP32,DFPlayer: Initialization
    ESP32->>ESP32: BLE Initialization<br/>GATT Server Start
    ESP32->>ESP32: DFPlayer Initialization<br/>microSD Read Preparation
    ESP32->>ESP32: BLE Advertising Start<br/>(Name: OBJ_001, etc.)

    Note over PC: Scan Start
    PC->>PC: BLE Scan Start<br/>(Continuous Monitoring)

    loop Periodic Scanning
        ESP32-->>PC: BLE Advertisement<br/>(Includes RSSI Info)
        PC->>PC: Device Name Filter<br/>(Starts with OBJ_)
        PC->>PC: RSSI Value Check
    end

    Note over User,PC: Object Approaches
    User->>ESP32: Bring Object Close to PC

    PC->>PC: RSSI >= Threshold<br/>(Example: -60 dBm)
    PC->>PC: Debounce Check<br/>(Time Passed Since Last Trigger)
    
    alt RSSI Threshold Exceeded & Debounce OK
        PC->>ESP32: GATT Connection Request
        ESP32-->>PC: Connection Established
        
        PC->>ESP32: Write to GATT Characteristic<br/>("PLAY" Command)
        
        par Parallel Processing
            ESP32->>ESP32: Check Playing Flag<br/>(isPlaying Check)
            ESP32->>ESP32: Cooldown Check<br/>(Time Since Last Playback)
            ESP32->>DFPlayer: Play Track 1 Command<br/>(dfplayer.play(1))
            DFPlayer->>DFPlayer: Load Audio from microSD<br/>(001.mp3)
            DFPlayer->>DFPlayer: Audio Playback Start
            DFPlayer-->>ESP32: BUSY Pin: LOW<br/>(Playing)
            ESP32->>ESP32: isPlaying = true
        and
            PC->>VLC: Video Playback Start<br/>(subprocess.Popen)
            VLC->>VLC: Load Video File
            VLC->>VLC: Video Playback Start
        end
        
        Note over ESP32,DFPlayer: During Playback
        loop Playback Monitoring
            ESP32->>ESP32: Monitor BUSY Pin State
            DFPlayer-->>ESP32: BUSY Pin State
        end
        
        DFPlayer->>DFPlayer: Audio Playback End
        DFPlayer-->>ESP32: BUSY Pin: HIGH<br/>(Playback Ended)
        ESP32->>ESP32: isPlaying = false
        
        VLC->>VLC: Video Playback End
        
        PC->>ESP32: GATT Disconnect
    end
```

## Error Handling and Guard Conditions

```mermaid
sequenceDiagram
    participant PC as Exhibition PC
    participant ESP32 as ESP32
    participant DFPlayer as DFPlayer Mini

    PC->>ESP32: GATT Connection Request
    
    alt Connection Failure
        ESP32-->>PC: Connection Error
        PC->>PC: Error Log Output<br/>Retry Connection
    end
    
    PC->>ESP32: Send "PLAY" Command
    
    alt Playing Flag is true
        ESP32->>ESP32: Ignore Command<br/>(Because Playing)
        Note over ESP32: Prevent Duplicate Playback
    end
    
    alt During Cooldown
        ESP32->>ESP32: Ignore Command<br/>(Insufficient Time Since Last Playback)
        Note over ESP32: Prevent Continuous Triggering
    end
    
    alt Normal Case
        ESP32->>DFPlayer: Playback Command
        DFPlayer-->>ESP32: Playback Start Success
    else DFPlayer Initialization Failure
        DFPlayer-->>ESP32: Error
        ESP32->>ESP32: Error Log Output<br/>(Retry Design Recommended)
    end
```

## Departure Detection Flow (C# Version Additional Feature)

```mermaid
sequenceDiagram
    participant PC as Exhibition PC<br/>(C# Version)
    participant ESP32 as ESP32
    participant VLC as VLC

    Note over PC: During Playback
    
    loop Periodic Departure Check
        PC->>PC: Timer Execution<br/>(1 Second Interval)
        PC->>PC: Check Last Detection Time
        
        alt Time Elapsed Since Last Detection<br/>(Example: 4 seconds)
            PC->>PC: Device Departure Detected
            PC->>ESP32: Send "STOP" Command<br/>(Optional)
            PC->>VLC: Stop Video
            VLC->>VLC: Playback Stopped
            PC->>PC: isPlaying = false<br/>isNear = false
        end
    end
```

## System Architecture Diagram

```mermaid
graph TB
    subgraph "Object Side"
        ESP32[ESP32<br/>BLE GATT Server]
        DFPlayer[DFPlayer Mini<br/>MP3 Player]
        SDCard[microSD<br/>Audio Files]
        Amp[Small Amplifier]
        Speaker[Wired Speaker]
        Battery[Battery]
        
        ESP32 -->|UART1| DFPlayer
        DFPlayer --> SDCard
        DFPlayer -->|BUSY Pin| ESP32
        DFPlayer --> Amp
        Amp --> Speaker
        Battery --> ESP32
        Battery --> DFPlayer
    end
    
    subgraph "Exhibition PC Side"
        PC[Exhibition PC]
        BLEAdapter[USB BLE Dongle<br/>or Built-in BLE]
        Python[Python Script<br/>Using bleak]
        VLC[VLC<br/>Video Playback]
        
        PC --> BLEAdapter
        PC --> Python
        PC --> VLC
    end
    
    ESP32 <-->|BLE Communication<br/>RSSI Monitoring| BLEAdapter
    BLEAdapter <--> Python
    Python -->|Trigger| VLC
```


