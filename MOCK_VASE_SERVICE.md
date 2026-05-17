# Mock SmartVase Service

## Overview
The Mock SmartVase Service emulates physical SmartVase device behavior for testing without actual hardware.

## Features

### 1. Ping Functionality
- **Endpoint**: `POST /api/mock-vase/{vaseId}/ping`
- **Success Rate**: 95% (simulates occasional network issues)
- **Response Time**: 10-100ms (simulated network latency)
- **Purpose**: Test vase connectivity from Admin Portal

### 2. Bouquet Image Management
- **Upload**: `POST /api/mock-vase/{vaseId}/bouquet-image`
- **Retrieve**: `GET /api/mock-vase/image/{vaseId}` (public, no auth)
- **Storage**: In-memory (ConcurrentDictionary)
- **Max Size**: 5MB
- **Formats**: JPG, PNG, GIF

### 3. Status Reporting
- **Endpoint**: `GET /api/mock-vase/{vaseId}/status`
- **Returns**: Online status, water level, temperature, humidity, bouquet image URL
- **Access**: Public (for Customer Portal)

### 4. Heartbeat Simulation
- **Endpoint**: `POST /api/mock-vase/{vaseId}/heartbeat`
- **Purpose**: Simulates periodic device check-ins
- **Access**: Public (in production, would use device auth)

## Usage in Admin Portal

### Vase Details Page
Navigate to: `http://localhost:5113/Vases/Details/{vaseId}`

**Features:**
1. **Device Information**: ID, status, health, connection
2. **Vendor Information**: Assigned vendor details
3. **Ping Test**: Click "Ping Vase" to test connectivity
4. **Bouquet Image Upload**: 
   - Select an image file
   - Preview before upload
   - Upload to mock service
   - Image immediately displayed after upload (cache-busted)

## Cache-Busting
Image URLs include timestamp query parameter: `?t={timestamp}`
- Ensures fresh image load after upload
- Prevents stale 404 cached responses

## Production Considerations
In production, this would be replaced with:
- Real IoT device firmware
- Blob storage (Azure Blob, AWS S3) for images
- MQTT or WebSocket for real-time communication
- Device-specific authentication tokens

