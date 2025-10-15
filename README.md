# BLE IMU Control - README

A web-based Bluetooth Low Energy (BLE) interface for controlling a pointer using IMU (Inertial Measurement Unit) data from ESP32 devices.

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

---

## 🔄 Current Data Manipulations

### Raw Data Processing

The application performs several manipulations on the raw IMU data:

1. **Header Validation**: Checks for 4-byte header `[0x41, 0x00, 0x06, 0x15]`
2. **Data Extraction**: Extracts YPR angles from bytes 28-35 (after header):
   - Yaw: `dataView.getFloat32(24, true)` (little-endian)
   - Pitch: `dataView.getFloat32(28, true)`
   - Roll: `dataView.getFloat32(32, true)`

3. **Range Clamping**:
   - Yaw: Limited to [-180°, +180°]
   - Pitch: Limited to [-90°, +90°]

4. **Calibration Offset Application**:
   ```javascript
   let clampedYaw = Math.max(-180, Math.min(180, yaw - yawOffset));
   let clampedPitch = Math.max(-90, Math.min(90, pitch - pitchOffset));
   ```

5. **Adaptive Sensitivity Scaling**:
   - Reduces sensitivity near extremes (>150° yaw, >75° pitch) to prevent edge-hugging
   - Auto-boosts underperforming axes (3x boost if one axis has <5° movement while other >10°)

6. **Non-linear Mapping** for large angles:
   ```javascript
   if (Math.abs(clampedYaw) > 120) {
       normalizedYaw = Math.sign(clampedYaw) * Math.sin((Math.abs(clampedYaw) / 180) * (Math.PI / 2));
   }
   ```

---

## ⚡ ESP32 Optimizations for Raw Data Usage

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

### 2. **Normalized Output Range**
```cpp
// Normalize to [-1.0, +1.0] range on ESP32
float normalizedYaw = constrain((currentYaw - yawOffset) / 180.0, -1.0, 1.0);
float normalizedPitch = constrain((currentPitch - pitchOffset) / 90.0, -1.0, 1.0);
```

### 3. **Simplified Data Packet Structure**
```cpp
struct SimpleIMUPacket {
    uint8_t header[2] = {0xFA, 0xCE};  // Simpler header
    float normalizedYaw;     // [-1.0 to +1.0]
    float normalizedPitch;   // [-1.0 to +1.0]
    float normalizedRoll;    // [-1.0 to +1.0]
    uint8_t checksum;
};
```

### 4. **Adaptive Sensitivity on ESP32**
```cpp
// Apply sensitivity adjustments on device
float applySensitivity(float angle, float baseAngle) {
    float sensitivity = 1.0;
    if (abs(angle) > 120) sensitivity = 0.3;  // Near extremes
    else if (abs(angle) > 60) sensitivity = 0.6;
    
    return angle * sensitivity;
}
```

---

## 📊 Visualization & Controls

### What's Being Plotted

The application visualizes:

1. **Red Pointer Dot**: Represents device orientation in 2D space
   - **X-axis**: Controlled by Yaw rotation (left/right device tilt)
   - **Y-axis**: Controlled by Pitch rotation (forward/backward device tilt)
   - **Container**: Bounded rectangular area with padding

2. **Real-time Status**: Shows current YPR angles and movement state
3. **Debug Panel**: Raw data analysis and movement calculations
4. **Inspector Panel**: Real-time data validation and edge detection

### Center Calibration System

**How it works:**
1. **Trigger**: User clicks "Calibrate Center" button
2. **Process**: 
   - Sets `isCalibrating = true` to pause movement updates
   - Captures current yaw/pitch as baseline offsets
   - Centers pointer visually to middle of container
   - Waits 1 second for device stabilization
3. **Result**: All future movements are relative to calibrated position

**Code Implementation:**
```javascript
function calibrateCenter() {
    isCalibrating = true;
    
    // Visual centering
    pointer.style.left = centerX + 'px';
    pointer.style.top = centerY + 'px';
    
    setTimeout(() => {
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

This system allows for precise control calibration without requiring device repositioning, making it ideal for various use cases from gaming to accessibility applications.

---

## 🚀 Getting Started

1. Open `index.html` in a modern browser (Chrome/Edge recommended)
2. Enable Bluetooth and ensure your ESP32 device is powered on
3. Click "Connect to Device" and select your FAN device from the list
4. Use "Calibrate Center" to set your preferred neutral position
5. Adjust sensitivity controls as needed for optimal response

## 📋 Requirements

- Modern web browser with Web Bluetooth API support
- ESP32 device with IMU sensor broadcasting on Heart Rate Service
- HTTPS connection (required for Web Bluetooth API)