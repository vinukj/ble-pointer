# BLE Cricket Bat Swing Analyzer - 3D Visualization

A comprehensive web-based Bluetooth Low Energy (BLE) interface for analyzing cricket bat swings using IMU (Inertial Measurement Unit) data from ESP32 devices. Features advanced 3D trajectory visualization, cricket-specific direction analysis, and precision timing controls.

## 🏏 Cricket Swing Analysis Features

### **3D Trajectory Visualization**
- **Real-time 3D rendering** using Three.js with WebGL acceleration
- **Swing trajectory tracking** with integrated accelerometer data
- **Speed-based visual feedback** - line thickness increases with bat speed
- **Height classification** - Lofted (green), Normal (white), Below (blue)
- **Automatic scaling** - keeps trajectories within view bounds

### **Cricket-Specific Direction Analysis**
- **Batsman's perspective view** - 0° = facing bowler, 90° = leg side, 270° = off side
- **Cricket field positions** - Mid-wicket, Square leg, Fine leg, Cover, Point, etc.
- **36-direction precision** with 10° increments for detailed shot analysis
- **Compass visualization** with real-time needle rotation

### **North Calibration System**
- **Calibrate North reference** - Set device orientation as "facing bowler"
- **Session-persistent calibration** - Retained across multiple shots and resets
- **Reference angle display** - Button shows calibrated angle: "🧭 North Set (3.3°)"
- **Default operation** - Works without calibration using 0° reference

### **Advanced Timing Controls**
- **Initial Run-up Delay** - Skip preparation phase (default 500ms, adjustable 0-2000ms)
- **Capture Window** - Precise analysis period (default 750ms, adjustable 250-2000ms)
- **Smart data collection** - Only analyzes ball impact zone, ignores follow-through
- **Phase-aware feedback** - "Run-up phase", "Analyzing swing", "Post-analysis"

### **Coordinate System Support**
- **NED Mode (Default ON)** - North-East-Down navigation frame
- **XYZ Sensor Frame** - Raw sensor coordinates
- **Toggle between systems** with live coordinate transformation

### **Weighted Direction Algorithm**
- **Impact detection** - Emphasizes high-acceleration moments (ball contact)
- **Speed + acceleration weighting** - Peak impact gets highest influence
- **Gravity compensation** - Removes 9.81 m/s² baseline for accurate analysis
- **Reduces noise** from preparation and follow-through movements

## 🔗 BLE Connection Overview

### How the Connection Works

The application establishes a BLE connection using the Web Bluetooth API with the following process:

1. **Device Discovery**: Uses `acceptAllDevices: true` to show all available Bluetooth devices in the browser picker
2. **Service Connection**: Connects to the Heart Rate Service (`0x180D`) as the primary communication channel
3. **Characteristic Binding**: Subscribes to Heart Rate Measurement characteristic (`0x2A37`) for IMU data notifications
4. **Data Stream Initialization**: Sends a trigger command `[0x03, 0x00, 0x06, 0x15, 0xE5]` to start IMU data transmission

### Connection Fallback Strategy
- **Primary**: Show all devices → User selects FAN device
- **Fallback 1**: Filter by namePrefix "FAN:"
- **Fallback 2**: Filter by Heart Rate Service UUID
- **Fallback 3**: Graceful error handling

---

## 🎮 Unity Implementation Code Snippet

Here's how to implement the same BLE connection in Unity (requires BLE plugins like "Bluetooth LE for iOS, tvOS and Android"):

```csharp
using System;
using UnityEngine;

public class BLEIMUController : MonoBehaviour
{
    // BLE Configuration
    private const string HEART_RATE_SERVICE_UUID = "180D";
    private const string HEART_RATE_MEASUREMENT_UUID = "2A37";
    private const string DEVICE_NAME_PREFIX = "FAN:";
    
    // IMU Data Variables
    private float yaw, pitch, roll;
    private float yawOffset = 0f, pitchOffset = 0f;
    
    void Start()
    {
        // Initialize BLE
        BluetoothLEHardwareInterface.Initialize(true, false, () => {
            Debug.Log("BLE Initialized");
            ScanForDevices();
        }, (error) => {
            Debug.LogError("BLE Error: " + error);
        });
    }
    
    void ScanForDevices()
    {
        BluetoothLEHardwareInterface.ScanForPeripheralsWithServices(null, (address, name) => {
            if (name.Contains("FAN"))
            {
                Debug.Log($"Found FAN device: {name}");
                ConnectToDevice(address);
            }
        }, null);
    }
    
    void ConnectToDevice(string deviceAddress)
    {
        BluetoothLEHardwareInterface.ConnectToPeripheral(deviceAddress, null, null, (address, serviceUUID, characteristicUUID) => {
            if (serviceUUID.ToUpper().Contains(HEART_RATE_SERVICE_UUID))
            {
                // Subscribe to IMU data notifications
                BluetoothLEHardwareInterface.SubscribeCharacteristicWithDeviceAddress(
                    deviceAddress, serviceUUID, characteristicUUID, null, OnIMUDataReceived);
                
                // Send trigger command
                byte[] triggerCommand = {0x03, 0x00, 0x06, 0x15, 0xE5};
                BluetoothLEHardwareInterface.WriteCharacteristic(deviceAddress, serviceUUID, characteristicUUID, triggerCommand, triggerCommand.Length, false, null);
            }
        });
    }
    
    void OnIMUDataReceived(string deviceAddress, string characteristic, byte[] data)
    {
        if (data.Length >= 68 && data[0] == 0x41 && data[1] == 0x00 && data[2] == 0x06 && data[3] == 0x15)
        {
            // Parse YPR data (offset 28-39 after 4-byte header)
            yaw = BitConverter.ToSingle(data, 28);
            pitch = BitConverter.ToSingle(data, 32);
            roll = BitConverter.ToSingle(data, 36);
            
            // Apply calibration and update game object
            UpdatePointerPosition();
        }
    }
    
    void UpdatePointerPosition()
    {
        float calibratedYaw = Mathf.Clamp(yaw - yawOffset, -180f, 180f);
        float calibratedPitch = Mathf.Clamp(pitch - pitchOffset, -90f, 90f);
        
        // Convert to Unity world coordinates
        Vector3 newPosition = new Vector3(
            calibratedYaw / 180f * maxRange,
            calibratedPitch / 90f * maxRange,
            0
        );
        
        transform.position = newPosition;
    }
    
    public void CalibrateCenter()
    {
        yawOffset = yaw;
        pitchOffset = pitch;
        Debug.Log($"Calibrated - Yaw: {yawOffset:F1}°, Pitch: {pitchOffset:F1}°");
    }
}
```

## 🏏 **How to Use the Cricket Swing Analyzer**

### **1. Setup and Connection**
1. **Connect BLE Device**: Click "Connect to FAN Device" and select your ESP32 device
2. **Verify NED Mode**: Ensure "📍 NED Mode: ON" is active (default)
3. **Calibrate North** (Recommended):
   - Hold bat facing the bowler
   - Click "🧭 Calibrate North"
   - Button shows reference angle: "🧭 North Set (3.3°)"

### **2. Configure Timing (Optional)**
- **Initial Run-up**: Set delay before analysis starts (default 500ms)
- **Capture Window**: Set analysis duration (default 750ms)
- **Click "⚙️ Update Timing"** to apply changes

### **3. Analyze Your Swings**
1. **Take your stance** facing the calibrated direction
2. **Start swing** - movement triggers automatic capture
3. **Watch real-time phases**:
   - "Run-up phase" → "Analyzing swing" → "Post-analysis"
4. **View results**: Direction, speed, swing height, and cricket field position

### **4. Interpret Results**
- **Direction**: Angle from batsman's perspective (0° = straight, 90° = leg side)
- **Cricket Position**: Field position names (Mid-wicket, Cover, Fine leg, etc.)
- **Swing Height**: Lofted (green), Normal (white), Below (blue)
- **Speed**: Average bat speed during impact zone

### **5. Reset for Next Shot**
- **Click "🔄 Reset"** to clear trajectory (calibration retained)
- **Ready for next swing** immediately

---

## 🔄 Advanced Data Processing

### **3D Trajectory Integration**
- **Double integration** of accelerometer data for position tracking
- **Gravity compensation** based on coordinate system (NED/XYZ)
- **Velocity damping** (0.95 factor) to simulate air resistance
- **Dynamic scaling** to keep trajectories within view bounds

### **Smart Direction Detection**
1. **Weighted Data Collection**:
   - Speed weighting (faster = more important)
   - Acceleration emphasis (2x weight for impact spikes)
   - Gravity baseline removal (subtracts 9.81 m/s²)

2. **Cricket Field Mapping**:
   ```
   0°-10°: Straight down ground
   30°-60°: Mid-wicket
   90°-120°: Fine leg
   270°-300°: Cover
   ```

3. **Batsman Perspective Conversion**:
   - Input: Device orientation relative to calibrated North
   - Output: Cricket field position from batsman's viewpoint

### **Timing Window Analysis**
```
Timeline: |--Run-up--|--Analysis--|--Post-Analysis--|
Duration: |  500ms   |   750ms    |     750ms       |
Action:   | (Ignore) | (Capture)  |   (Ignore)      |
```

---

## ⚡ ESP32 Optimizations for Enhanced Performance

To use the data directly without client-side manipulations, consider these ESP32 modifications:

### 1. **Pre-Calibrated Data Transmission**
```cpp
// On ESP32 - apply calibration before sending
struct CalibratedIMUData {
    float calibratedYaw;    // Already offset-corrected
    float calibratedPitch;  // Already offset-corrected
    float calibratedRoll;   // Already offset-corrected
    uint8_t movingFlag;
    uint16_t timestamp;
};

// Send calibration command from client
void handleCalibrationCommand() {
    yawOffset = currentYaw;
    pitchOffset = currentPitch;
    // Store in EEPROM for persistence
    EEPROM.put(0, yawOffset);
    EEPROM.put(4, pitchOffset);
}
```

## 🎯 **File Structure**

### **Main Files**
- **`index-3d.html`** - Cricket Swing Analyzer with 3D visualization
- **`index.html`** - Basic 2D pointer control interface  
- **`README.md`** - This documentation

### **index-3d.html Features**
- 3D trajectory visualization using Three.js
- Cricket-specific swing analysis
- Advanced timing controls
- North calibration system
- Real-time direction feedback

### **index.html Features**  
- Simple 2D pointer movement
- Basic YPR visualization
- Center calibration
- Movement threshold detection

---

## � **Technical Requirements**

### **Browser Support**
- **Chrome/Edge**: Full Web Bluetooth API support
- **Firefox**: Experimental support (enable in about:config)
- **Safari**: Limited support (iOS 16+)
- **HTTPS Required**: Web Bluetooth only works over secure connections

### **Device Requirements**
- **ESP32** with BLE capabilities
- **IMU sensor** (accelerometer + gyroscope)
- **Heart Rate Service** implementation (UUID: 0x180D)
- **Data transmission** via characteristic 0x2A37

### **Performance Specifications**
- **Update Rate**: ~25Hz (40ms intervals)
- **Latency**: <50ms end-to-end
- **Precision**: ±0.1° angular resolution
- **Range**: ±180° yaw, ±90° pitch

---

## 🏏 **Cricket Analysis Algorithms**

### **Direction Calculation**
```javascript
// Convert velocity vector to cricket field angle
function getDirection36(velocity) {
    let angleRad = Math.atan2(velocity.y, velocity.x); // NED coordinates
    let angleDeg = (angleRad * 180 / Math.PI + 360) % 360;
    let calibratedAngle = (angleDeg + compassOffset) % 360;
    return { degrees: calibratedAngle, direction: cricketFieldName };
}
```

### **Weighted Analysis**
```javascript
// Emphasize impact moments
const speedWeight = currentSpeed;
const accelWeight = Math.max(0, accelMagnitude - 9.81);
const combinedWeight = speedWeight + (accelWeight * 2);
```

### **Cricket Field Mapping**
- **0°-30°**: Straight shots (down the ground)
- **30°-90°**: Leg side (mid-wicket to square leg)
- **90°-180°**: Deep leg side (fine leg to backward)
- **180°-270°**: Off side back (third man to point)
- **270°-360°**: Off side front (cover to mid-off)

---

## 📈 **Data Packet Structure**
        // Capture current orientation as "center"
        yawOffset = lastYaw;
        pitchOffset = lastPitch;
        isCalibrating = false;
    }, 1000);
}
```

### Sensitivity Controls

- **Yaw Boost**: 3x multiplier for horizontal movement sensitivity
- **Pitch Boost**: 3x multiplier for vertical movement sensitivity
- **Adaptive Scaling**: Automatic sensitivity reduction near container edges
- **Auto-boost**: Compensates for underperforming movement axes

### **Expected Data Format**
```
Packet Structure (90 bytes total):
├── BLE Header: [0x37, 0x00, 0x06, 0x0F]
├── Embedded Header: [0x41, 0x00, 0x06, 0x15] 
├── Position Data: p[3] (12 bytes)
├── Velocity Data: v[3] (12 bytes)  
├── YPR Angles: yaw/pitch/roll (12 bytes)
├── Acceleration: accel[3] (12 bytes)
└── Movement Flag: movingFlag (1 byte)
```

### **Data Extraction Points**
- **Yaw**: Bytes 28-31 (float32, little-endian)
- **Pitch**: Bytes 32-35 (float32, little-endian)  
- **Roll**: Bytes 36-39 (float32, little-endian)
- **Acceleration**: Bytes 40-51 (3x float32)
- **Movement Flag**: Byte 40 (1 = moving, 0 = stationary)

---

## 🚀 **Getting Started**

### **Quick Setup**
1. **Clone repository** or download files
2. **Serve over HTTPS** (required for Web Bluetooth):
   ```bash
   python3 -m http.server 8000 --bind 0.0.0.0
   # Access via https://localhost:8000/index-3d.html
   ```
3. **Connect ESP32 device** with IMU sensor
4. **Open in Chrome/Edge browser**
5. **Click "Connect to FAN Device"**

### **Cricket Swing Analysis Workflow**
1. **Calibrate North** - Point device toward bowler, click calibrate
2. **Adjust timing** - Set run-up delay and capture window (optional)
3. **Take your stance** - Hold device/bat in playing position
4. **Start swinging** - Movement automatically triggers capture
5. **View analysis** - Direction, speed, height, and field position
6. **Reset for next shot** - Click reset (keeps calibration)

### **Troubleshooting**
- **No device found**: Ensure ESP32 is advertising with Heart Rate Service
- **Connection fails**: Check device is not connected to other apps
- **No data**: Verify trigger command is being sent correctly
- **Inaccurate directions**: Recalibrate North reference
- **Timing issues**: Adjust run-up delay and capture window

---

## 📝 **Development Notes**

### **Future Enhancements**
- **Shot classification** (drive, cut, pull, sweep, etc.)
- **Ball speed estimation** from impact acceleration
- **Swing plane analysis** (bat path visualization)
- **Historical shot tracking** and session statistics
- **Export data** to CSV/JSON for external analysis

### **Known Limitations**
- **Web Bluetooth support** varies by browser/OS
- **HTTPS requirement** for secure contexts only
- **Single device connection** at a time
- **Timing precision** depends on BLE update rate

This cricket swing analyzer provides professional-level biomechanical analysis using consumer-grade hardware, making advanced sports analytics accessible to players at all levels.
5. Adjust sensitivity controls as needed for optimal response

## 📋 Requirements

- Modern web browser with Web Bluetooth API support
- ESP32 device with IMU sensor broadcasting on Heart Rate Service
- HTTPS connection (required for Web Bluetooth API)