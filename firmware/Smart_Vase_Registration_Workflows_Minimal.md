# Smart Vase Registration Workflows - Implementation Guide

## Executive Summary

This document outlines three registration workflow options for Smart Vase devices that start without network configuration. Each approach is presented with high-level logic and TODO placeholders for actual implementation.

---

## Table of Contents

1. [Current Device State](#current-device-state)
2. [Registration Options Overview](#registration-options-overview)
3. [Option 1: WiFi Provisioning + QR Code (Recommended)](#option-1-wifi-provisioning--qr-code)
4. [Option 2: USB Serial Configuration](#option-2-usb-serial-configuration)
5. [Option 3: Hybrid Approach (Best Practice)](#option-3-hybrid-approach)
6. [QR Code Data Formats](#qr-code-data-formats)
7. [Implementation Roadmap](#implementation-roadmap)
8. [Testing Checklist](#testing-checklist)

---

## Current Device State

### On First Power-On

**Available:**
- ✅ Device ID (from MAC address: `ESP32-40C86C`)
- ✅ LCD display with LVGL
- ✅ Sensors (DHT11/22, IR obstacle sensor)
- ✅ QR code generation capability

**Missing:**
- ❌ WiFi credentials
- ❌ MQTT server settings
- ❌ Vendor assignment
- ❌ Backend registration

---

## Registration Options Overview

### Comparison Matrix

| Feature | WiFi Provisioning | Serial Config | Hybrid |
|---------|------------------|---------------|--------|
| **Scalability** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐ Poor | ⭐⭐⭐⭐⭐ Excellent |
| **User Experience** | ⭐⭐⭐⭐⭐ Professional | ⭐⭐ Technical | ⭐⭐⭐⭐⭐ Best |
| **Time per Device** | ~2-3 minutes | ~5-10 minutes | ~2-3 minutes |
| **Implementation** | ⭐⭐⭐ Complex | ⭐⭐⭐⭐⭐ Simple | ⭐⭐ Very Complex |
| **Hardware Needed** | Smartphone/Tablet | USB Cable + Laptop | Smartphone or USB |
| **Debugging** | ⭐⭐ Limited | ⭐⭐⭐⭐⭐ Full Access | ⭐⭐⭐⭐⭐ Full Access |

**Recommendation**: Start with **Option 3 (Hybrid)** - implement serial config first for development, add WiFi provisioning for production.

---

## Option 1: WiFi Provisioning + QR Code

### Workflow Overview

```
Device Powers On (No WiFi)
    ↓
Creates WiFi AP: "SmartVase-XXXXXX"
    ↓
Technician connects phone to AP
    ↓
Captive portal opens (192.168.4.1)
    ↓
Technician selects shop WiFi + enters password
    ↓
Device saves credentials & restarts
    ↓
Device connects to shop WiFi
    ↓
Device displays QR code for registration
    ↓
Admin scans QR → selects vendor → submits
    ↓
Backend sends MQTT config to device
    ↓
Device receives config → saves → REGISTERED ✓
```

### Phase 1: WiFi Configuration

#### Device Setup Mode

```cpp
void startSetupMode() {
    // TODO: Create WiFi Access Point
    // AP Name: "SmartVase-" + last 6 chars of device ID
    // Password: "flowershop" (configurable)
    
    // TODO: Start DNS server for captive portal (port 53)
    
    // TODO: Start web server on port 80
    
    // TODO: Update LCD to show connection instructions
    
    Serial.println("Setup mode active - AP: SmartVase-XXXXXX");
}
```

#### Captive Portal Web Server

```cpp
void startConfigServer() {
    // TODO: Implement route: GET /
    // Serve HTML form with:
    //   - WiFi network dropdown (populated via AJAX)
    //   - Password input field
    //   - Submit button
    
    // TODO: Implement route: GET /scan
    // Return JSON array of available WiFi networks:
    // [{"ssid": "ShopWiFi", "rssi": -45, "secure": true}, ...]
    
    // TODO: Implement route: POST /configure
    // Accept: ssid, password
    // Actions:
    //   1. Save credentials to NVS
    //   2. Return success HTML page
    //   3. Schedule device restart (3 seconds)
    
    // TODO: Implement route: ALL /* (catch-all)
    // Redirect to / for captive portal
}
```

#### WiFi Connection

```cpp
void connectToWiFi() {
    // TODO: Load saved WiFi credentials from NVS
    
    // TODO: Set WiFi mode to STA
    
    // TODO: Begin connection with timeout (20 attempts)
    
    // TODO: If successful:
    //   - Set wifiConnected = true
    //   - Initialize NTP time
    //   - Update LCD
    
    // TODO: If failed:
    //   - Re-enter setup mode
}
```

### Phase 2: Vendor Registration

#### QR Code Display

```cpp
void showRegistrationScreen() {
    // TODO: Generate QR code with registration data
    // Format: URL or JSON (see QR Code Formats section)
    
    // TODO: Draw QR code on LCD (center of screen)
    
    // TODO: Display Device ID below QR code
    
    // TODO: Show instruction text: "Scan to register"
}
```

#### Backend Registration API

```csharp
// C# Controller
[HttpPost("api/vases/register")]
public async Task<IActionResult> RegisterVase(RegisterVaseDto dto)
{
    // TODO: Validate device ID format (ESP32-XXXXXX)
    
    // TODO: Check device not already registered
    
    // TODO: Validate vendor exists and user has permission
    
    // TODO: Create SmartVase database record
    
    // TODO: Generate MQTT credentials for device
    
    // TODO: Publish MQTT config to device:
    //   Topic: vases/{deviceId}/config
    //   Payload: {
    //     action: "registration_confirmed",
    //     vendorId, vendorName, locationName,
    //     mqttBroker, mqttPort, mqttUser, mqttPassword,
    //     heartbeatInterval
    //   }
    
    // TODO: Return success response
}
```

#### Device Config Handler

```cpp
void handleConfig(StaticJsonDocument<512> &doc) {
    if (doc["action"] == "registration_confirmed") {
        // TODO: Extract and save:
        //   - Vendor ID
        //   - MQTT broker settings
        //   - Heartbeat interval
        
        // TODO: Update state to REGISTERED
        
        // TODO: Save configuration to NVS
        
        // TODO: Play confirmation beeps (3x)
        
        // TODO: Update LCD to "Ready for Bouquet"
        
        // TODO: Send acknowledgment via MQTT
    }
}
```

### Advantages
- ✅ Professional, scalable
- ✅ No USB cables needed
- ✅ Mobile-friendly
- ✅ Industry standard approach

### Disadvantages
- ⚠️ More complex to implement
- ⚠️ Requires web server libraries
- ⚠️ Higher memory usage

---

## Option 2: USB Serial Configuration

### Workflow Overview

```
Connect USB cable to device
    ↓
Open serial terminal (115200 baud)
    ↓
Type: "setup"
    ↓
Enter WiFi SSID + password
    ↓
Enter MQTT server + credentials
    ↓
Enter Vendor ID
    ↓
Device saves configuration
    ↓
Device connects & operates normally
```

### Implementation

#### Serial Menu System

```cpp
void showConfigMenu() {
    // TODO: Display menu header with device ID
    
    // TODO: Show current settings (WiFi, MQTT, Vendor)
    
    // TODO: Display numbered options:
    //   1. Configure WiFi
    //   2. Configure MQTT
    //   3. Set Vendor ID
    //   4. Test Connection
    //   5. Show Status
    //   6. Factory Reset
    //   7. Save & Exit
    
    // TODO: Wait for user input
    
    // TODO: Call appropriate handler based on selection
}
```

#### WiFi Configuration (Serial)

```cpp
void configureWiFiSerial() {
    // TODO: Scan for available networks
    
    // TODO: Display networks with signal strength
    
    // TODO: Prompt for SSID (or number from list)
    
    // TODO: Prompt for password (hidden)
    
    // TODO: Test connection
    
    // TODO: Save if successful
}
```

#### MQTT Configuration (Serial)

```cpp
void configureMQTTSerial() {
    // TODO: Prompt for MQTT broker hostname/IP
    
    // TODO: Prompt for port (default: 1883)
    
    // TODO: Prompt for username
    
    // TODO: Prompt for password (hidden)
    
    // TODO: Save configuration
}
```

#### Vendor Assignment (Serial)

```cpp
void configureVendorSerial() {
    // TODO: Prompt for Vendor ID (GUID format)
    
    // TODO: Validate GUID format
    
    // TODO: Prompt for confirmation
    
    // TODO: Set isRegistered = true
    
    // TODO: Save configuration
}
```

### Advantages
- ✅ Simple to implement
- ✅ Full debugging access
- ✅ No network required
- ✅ Direct control

### Disadvantages
- ⚠️ Not scalable (one at a time)
- ⚠️ Requires laptop + USB cable
- ⚠️ Manual typing (error-prone)
- ⚠️ Time-consuming

---

## Option 3: Hybrid Approach

### Startup Logic

```cpp
void setup() {
    // TODO: Initialize hardware (display, sensors, LVGL)
    
    // TODO: Generate device ID from MAC
    
    // TODO: Load saved configuration from NVS
    
    Serial.println("Smart Vase v1.0.0");
    
    // ===== PRIORITY 1: Serial Setup Override =====
    Serial.println("Press 's' for serial setup (5 seconds)...");
    
    // TODO: Wait 5 seconds, check for 's' key
    
    if (serialSetupRequested) {
        // TODO: Enter serial configuration menu
        // (See Option 2 implementation)
    }
    
    // ===== PRIORITY 2: Check WiFi Configuration =====
    if (strlen(wifi_ssid) == 0) {
        // TODO: No WiFi configured
        // Start WiFi provisioning mode (See Option 1)
    }
    
    // ===== PRIORITY 3: Connect to WiFi =====
    // TODO: Attempt connection to configured WiFi
    
    if (!wifiConnected) {
        // TODO: Connection failed - restart setup mode
    }
    
    // ===== PRIORITY 4: Check Registration =====
    if (!isRegistered) {
        // TODO: Show QR code for vendor assignment
        // Wait for MQTT registration message
    }
    
    // ===== PRIORITY 5: Connect MQTT =====
    if (strlen(mqtt_server) > 0) {
        // TODO: Setup and connect MQTT client
    }
    
    // TODO: Begin normal operation
}
```

### Decision Tree

```
Power On
    │
    ├─→ [5sec] 's' pressed? ──YES──→ Serial Setup Menu
    │                         NO
    │                         ↓
    ├─→ WiFi configured? ──NO──→ Start WiFi Provisioning AP
    │                     YES
    │                     ↓
    ├─→ Connect WiFi ──SUCCESS──→ Continue
    │                 FAILED
    │                 ↓
    │                 Start WiFi Provisioning AP
    │
    ├─→ Registered? ──NO──→ Show QR Code (wait for MQTT)
    │                YES
    │                ↓
    ├─→ Connect MQTT
    │
    └─→ Normal Operation (sensors, heartbeat, bouquet detection)
```

### Advantages
- ✅ Best of both worlds
- ✅ Production-ready (WiFi provisioning)
- ✅ Debug-friendly (serial access)
- ✅ Flexible configuration paths
- ✅ Resilient (multiple fallbacks)

### Disadvantages
- ⚠️ Most complex to implement
- ⚠️ Multiple code paths to test
- ⚠️ Higher memory usage

---

## QR Code Data Formats

### Format 1: Simple Device ID

**Use Case**: Fallback, manual entry required

```cpp
void generateSimpleQR() {
    char qrData[32];
    // TODO: Format: "ESP32-40C86C"
    snprintf(qrData, sizeof(qrData), "%s", device_id);
    
    // TODO: Generate QR code with qrcode library
}
```

**Result**: `ESP32-40C86C`

---

### Format 2: Registration URL

**Use Case**: Browser-friendly, direct scanning

```cpp
void generateRegistrationURLQR() {
    char qrData[256];
    
    // TODO: Build URL with query parameters
    snprintf(qrData, sizeof(qrData),
        "https://flowershop.app/register?"
        "deviceId=%s&mac=%s&ip=%s&fw=1.0.0",
        device_id,
        WiFi.macAddress().c_str(),
        WiFi.localIP().toString().c_str()
    );
    
    // TODO: Generate QR code
}
```

**Result**: `https://flowershop.app/register?deviceId=ESP32-40C86C&mac=24:6F:28:40:C8:6C&ip=192.168.1.45&fw=1.0.0`

---

### Format 3: JSON Payload

**Use Case**: Mobile app, structured data

```cpp
void generateJSONQR() {
    char qrData[512];
    
    // TODO: Build JSON object
    StaticJsonDocument<256> doc;
    doc["action"] = "register_vase";
    doc["deviceId"] = device_id;
    doc["mac"] = WiFi.macAddress();
    doc["ip"] = WiFi.localIP().toString();
    doc["firmware"] = "1.0.0";
    
    // TODO: Serialize to string
    serializeJson(doc, qrData, sizeof(qrData));
    
    // TODO: Generate QR code
}
```

**Result**: 
```json
{
  "action": "register_vase",
  "deviceId": "ESP32-40C86C",
  "mac": "24:6F:28:40:C8:6C",
  "ip": "192.168.1.45",
  "firmware": "1.0.0"
}
```

---

### Format 4: Deep Link (Native App)

**Use Case**: Launch mobile app directly

```cpp
void generateDeepLinkQR() {
    char qrData[256];
    
    // TODO: Format custom URI scheme
    snprintf(qrData, sizeof(qrData),
        "flowershop://register/%s?mac=%s",
        device_id,
        WiFi.macAddress().c_str()
    );
    
    // TODO: Generate QR code
}
```

**Result**: `flowershop://register/ESP32-40C86C?mac=24:6F:28:40:C8:6C`

---

## Implementation Roadmap

### Week 1: Serial Configuration (MVP)
**Goal**: Get basic configuration working via USB

**Tasks**:
- [ ] Implement `processSerialCommands()` function
- [ ] Implement `showConfigMenu()` with numbered options
- [ ] Implement `configureWiFiSerial()` - scan & connect
- [ ] Implement `configureMQTTSerial()` - server settings
- [ ] Implement `configureVendorSerial()` - GUID entry
- [ ] Implement `showStatus()` - display all settings
- [ ] Implement `factoryReset()` - clear NVS
- [ ] Test end-to-end configuration flow

**Deliverable**: Device configurable via serial terminal

---

### Week 2: WiFi Provisioning Portal
**Goal**: Professional captive portal setup

**Tasks**:
- [ ] Add libraries: `ESPAsyncWebServer`, `DNSServer`
- [ ] Implement `startSetupMode()` - create AP
- [ ] Implement `startConfigServer()` - web routes
- [ ] Create HTML/CSS for captive portal form
- [ ] Implement `/scan` endpoint - WiFi network list
- [ ] Implement `/configure` endpoint - save credentials
- [ ] Add DNS captive portal redirect
- [ ] Implement auto-restart after configuration
- [ ] Test with multiple devices

**Deliverable**: Device configurable via WiFi AP + web portal

---

### Week 3: QR-Based Vendor Assignment
**Goal**: Backend integration for registration

**Tasks**:
- [ ] Generate registration QR codes (choose format)
- [ ] Display QR on LCD when connected but unregistered
- [ ] Implement backend API: `POST /api/vases/register`
- [ ] Implement MQTT config publishing from backend
- [ ] Implement device MQTT config handler
- [ ] Add registration confirmation (buzzer + LCD)
- [ ] Test registration flow end-to-end
- [ ] Add error handling for registration failures

**Deliverable**: Complete registration workflow

---

### Week 4: Mobile App Integration (Optional)
**Goal**: Native mobile app for QR scanning

**Tasks**:
- [ ] Choose mobile framework (React Native / Flutter)
- [ ] Implement QR scanner with camera
- [ ] Parse QR data (URL / JSON / Deep Link)
- [ ] Auto-fill registration form
- [ ] Submit to backend API
- [ ] Show success/error feedback
- [ ] Test on iOS and Android

**Deliverable**: Mobile app for streamlined registration

---

## Testing Checklist

### WiFi Provisioning Tests

- [ ] Device creates AP on first boot
- [ ] AP SSID format: `SmartVase-XXXXXX`
- [ ] AP password works
- [ ] Captive portal opens automatically
- [ ] Network scan returns WiFi list
- [ ] WiFi connection successful with valid credentials
- [ ] WiFi connection fails gracefully with invalid credentials
- [ ] Credentials saved to NVS
- [ ] Device restarts after configuration
- [ ] Device connects to WiFi automatically after restart
- [ ] AP mode shuts down after successful connection

### Serial Configuration Tests

- [ ] Serial menu accessible via `setup` command
- [ ] WiFi scan shows available networks
- [ ] WiFi configuration saves and works
- [ ] MQTT configuration saves correctly
- [ ] Vendor ID assignment works
- [ ] Status display shows accurate information
- [ ] Factory reset erases all configuration
- [ ] Device persists settings after power cycle

### Registration Tests

- [ ] QR code displays when WiFi connected but not registered
- [ ] QR code scannable with phone camera
- [ ] QR data format correct (URL / JSON / Deep Link)
- [ ] Admin portal receives registration request
- [ ] Backend validates device ID format
- [ ] Backend prevents duplicate registration
- [ ] Backend validates vendor permissions
- [ ] MQTT config published to correct topic
- [ ] Device receives MQTT config message
- [ ] Device saves MQTT credentials
- [ ] Device transitions to REGISTERED state
- [ ] Buzzer plays confirmation (3 beeps)
- [ ] LCD updates to "Ready for Bouquet"

### Persistence Tests

- [ ] Configuration survives power cycle
- [ ] WiFi auto-reconnects on boot
- [ ] MQTT auto-reconnects on boot
- [ ] Registration status persists
- [ ] Factory reset clears everything
- [ ] NVS partition not corrupted after many writes

### Error Handling Tests

- [ ] Invalid WiFi password shows error
- [ ] WiFi connection timeout handled
- [ ] MQTT connection failure handled
- [ ] Network loss triggers offline mode
- [ ] Network recovery triggers reconnection
- [ ] Invalid QR data handled gracefully
- [ ] Backend API errors shown to admin
- [ ] Device continues functioning if backend unavailable

---

## Configuration Storage (NVS)

### Preferences Keys

```cpp
// NVS namespace: "vase-config"

String wifi_ssid;          // Key: "wifi_ssid"     (max 32 chars)
String wifi_password;      // Key: "wifi_pass"     (max 64 chars)
String mqtt_server;        // Key: "mqtt_server"   (max 32 chars)
int mqtt_port;             // Key: "mqtt_port"     (default: 1883)
String mqtt_user;          // Key: "mqtt_user"     (max 32 chars)
String mqtt_password;      // Key: "mqtt_pass"     (max 64 chars)
String shop_id;            // Key: "shop_id"       (GUID, max 40 chars)
bool isRegistered;         // Key: "registered"    (boolean)
```

### Load Configuration

```cpp
void loadConfiguration() {
    preferences.begin("vase-config", true); // Read-only
    
    // TODO: Load all settings from NVS using keys above
    
    preferences.end();
    
    // TODO: Log loaded configuration to Serial
}
```

### Save Configuration

```cpp
void saveConfiguration() {
    preferences.begin("vase-config", false); // Read-write
    
    // TODO: Save all settings to NVS using keys above
    
    preferences.end();
    
    Serial.println("✓ Configuration saved");
}
```

### Factory Reset

```cpp
void factoryReset() {
    // TODO: Clear entire NVS namespace
    preferences.begin("vase-config", false);
    preferences.clear();
    preferences.end();
    
    // TODO: Restart device
    ESP.restart();
}
```

---

## MQTT Topics Structure

### Device-Specific Topics

```
vases/{deviceId}/config          # Backend → Device (registration config)
vases/{deviceId}/commands        # Backend → Device (bouquet commands)
vases/{deviceId}/status          # Device → Backend (status updates)
vases/{deviceId}/heartbeat       # Device → Backend (health check)
vases/{deviceId}/sensors         # Device → Backend (sensor data)
vases/{deviceId}/bouquets        # Device ↔ Backend (bouquet events)
```

### Global Topics

```
vases/register                   # Device → Backend (initial registration)
vases/discovery                  # Backend → All Devices (announcements)
```

### QoS Levels

```
config:     QoS 2 (exactly once) + retained
commands:   QoS 1 (at least once)
status:     QoS 0 (at most once)
heartbeat:  QoS 0 (at most once)
sensors:    QoS 0 (at most once)
bouquets:   QoS 1 (at least once)
```

---

## Security Considerations

### WiFi Provisioning Security

```cpp
// TODO: Implement WPA2 encryption for setup AP
// Default password: "flowershop" (should be configurable)

// TODO: Add timeout for setup mode (30 minutes)
// Auto-restart if no configuration received

// TODO: Rate limit configuration attempts
// Prevent brute force password attacks
```

### MQTT Security

```cpp
// TODO: Use TLS/SSL for MQTT connection (port 8883)

// TODO: Generate unique credentials per device

// TODO: Implement certificate validation

// TODO: Rotate credentials periodically
```

### Registration Security

```cpp
// TODO: Add timestamp to QR code (expire after 10 minutes)

// TODO: Add checksum/signature to QR data

// TODO: Validate MAC address matches device

// TODO: Prevent replay attacks
```

---

## Libraries Required

### ESP32 Platform

```ini
; platformio.ini

[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

lib_deps = 
    # Display & UI
    TFT_eSPI
    lvgl/lvgl @ ^8.3.0
    
    # Sensors
    adafruit/DHT sensor library
    
    # Networking
    ESP Async WebServer           # WiFi provisioning portal
    DNSServer                     # Captive portal redirect
    PubSubClient                  # MQTT client
    
    # QR Code Generation
    ricmoo/QRCode @ ^0.0.1
    
    # JSON
    ArduinoJson @ ^6.21.0
    
    # Storage
    Preferences                   # Built-in NVS library
```

---

## Next Steps

### Immediate Actions

1. **Choose Implementation Strategy**
   - Start with Option 2 (Serial) for quick development
   - Add Option 1 (WiFi Provisioning) for production
   - Combine into Option 3 (Hybrid) for final product

2. **Set Up Development Environment**
   - Install required libraries
   - Configure platformio.ini
   - Test compilation

3. **Implement Core Functions**
   - Configuration storage (NVS)
   - WiFi connection logic
   - MQTT client setup
   - Serial command handler

4. **Build Frontend**
   - Captive portal HTML/CSS/JS
   - Admin registration page
   - Mobile app (optional)

5. **Backend Integration**
   - Registration API endpoint
   - MQTT broker configuration
   - Credential generation

### Development Priorities

**Phase 1 (Week 1)**: Serial Configuration
- Goal: Manual configuration working
- Deliverable: Device configurable via USB

**Phase 2 (Week 2)**: WiFi Provisioning
- Goal: Professional setup experience
- Deliverable: Captive portal working

**Phase 3 (Week 3)**: QR Registration
- Goal: End-to-end registration flow
- Deliverable: Backend integration complete

**Phase 4 (Week 4)**: Polish & Testing
- Goal: Production-ready system
- Deliverable: Deployed to first shop

---

## Support & Troubleshooting

### Common Issues

**Issue**: Device won't connect to WiFi
- Check SSID/password correct
- Verify 2.4GHz network (not 5GHz only)
- Check router MAC filtering
- Try serial configuration to test

**Issue**: Captive portal doesn't open
- Check DNS server running
- Verify phone connected to device AP
- Manually navigate to 192.168.4.1
- Check firewall settings

**Issue**: QR code won't scan
- Increase QR code scale (larger modules)
- Ensure good lighting
- Try different QR scanner app
- Check QR data not too long (use shorter format)

**Issue**: MQTT connection fails
- Verify broker hostname/IP correct
- Check port (1883 or 8883)
- Validate credentials
- Test with MQTT client tool (MQTT Explorer)

**Issue**: Registration doesn't complete
- Check backend API endpoint accessible
- Verify device receives MQTT config
- Check topic subscription successful
- Review backend logs for errors

---

## Conclusion

This document provides a complete blueprint for implementing smart vase registration workflows. Start with the simplest approach (serial configuration) to validate the concept, then progressively add WiFi provisioning and QR-based registration for a production-ready system.

The hybrid approach gives you maximum flexibility during development while providing a professional user experience in production deployment.

**Recommended Path**:
1. Implement serial configuration (Week 1)
2. Add WiFi provisioning (Week 2)  
3. Integrate QR registration (Week 3)
4. Deploy and iterate (Week 4+)

Good luck with your implementation! 🌸
