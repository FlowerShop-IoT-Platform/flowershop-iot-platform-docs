 # Complete FlowerShop IoT Platform Workflow

> **Legacy / hardware-free testing walkthrough.** This describes the end-to-end flow using the
> **mock vase service** (in-memory image storage) for testing without physical hardware. In
> production, bouquet photos upload to `POST /api/v1/bouquets/{id}/photo` and are stored in
> **Cloudflare R2** (served from `img.findmyflowers.pl`), and vases communicate over **MQTT/TLS
> (HiveMQ Cloud)**. Use this document only for the no-hardware demo path.

## Services Overview

| Service | Local URL | Prod URL | Purpose |
|---------|-----------|----------|---------|
| **API Backend** | http://localhost:8080 | https://api.findmyflowers.pl | Core business logic, data persistence |
| **Admin Portal** | http://localhost:5001 | https://admin.findmyflowers.pl | Platform administration, vendor/vase management |
| **Vendor Portal** | http://localhost:5002 | https://vendors.findmyflowers.pl | Vendor dashboard, bouquet/vase management |
| **Customer App** | http://localhost:5003 | https://app.findmyflowers.pl | Customer browsing, bouquet discovery |

## Complete Workflow: From Vase to Customer

### 1. **Admin Creates Vendor** (Admin Portal)
- Navigate to: http://localhost:5001/Vendors/Create
- Create a vendor (FlowerShop or FreelanceFlorist)
- Set vendor location with address and coordinates
- **Result**: Vendor registered and active

### 2. **Admin Registers SmartVase** (Admin Portal)
- Navigate to: http://localhost:5001/Vases/Register
- Enter Device ID (format: ESP32-XXXXXX)
- Select vendor from list
- **Result**: Vase assigned to vendor at vendor's location

### 3. **Upload Bouquet Image** (Admin or Vendor Portal)

**Option A: Admin Portal**
- Navigate to: http://localhost:5001/Vases/Details/{vaseId}
- Click "Upload Image"
- Select bouquet photo
- **Result**: Image stored in mock vase service

**Option B: Vendor Portal**
- Navigate to: http://localhost:5002/Vases
- Click eye icon on any vase
- Upload bouquet photo
- **Result**: Image stored AND bouquet entity created automatically

### 4. **Mock Vase Service Processes Image**
- **Stores**: Image in memory (ConcurrentDictionary)
- **Creates**: Bouquet entity in database
- **Links**: Bouquet to vase with image URL
- **Image URL**: `http://localhost:8080/api/mock-vase/image/{vaseId}`

### 5. **Customer Views Bouquet** (Customer Portal)
- Navigate to: http://localhost:5003
- See bouquets on map within 2km radius (default)
- **Map Marker**: Round 50px circular icon with bouquet image
- **Click Marker**: Popup shows full image, price, vendor, distance
- **Image Source**: Loaded from mock vase service

## Key Features Implemented

### Admin Portal
✅ Vendor creation and management
✅ Vase registration with device ID validation
✅ Vase details page with ping functionality
✅ Bouquet image upload
✅ Location editing with geocoding
✅ Pending installations tracking

### Vendor Portal  
✅ Dashboard with statistics (active vases, bouquets, orders)
✅ My Vases page showing:
  - Vendor's location for all vases
  - Current bouquet thumbnail (40px circle)
  - Last heartbeat status
✅ Vase details page with image upload
✅ Action buttons:
  - View vase details
  - Upload bouquet photo
  - Create new bouquet

### Customer Portal
✅ Interactive map with 2km default radius
✅ Bouquet count badge at center top
✅ Round bouquet image markers (50px)
✅ Detailed popups with images
✅ Bouquet images loaded from vases
✅ Distance calculation from user location

### Mock SmartVase Service
✅ Ping endpoint (95% success rate)
✅ Image upload (max 5MB, JPG/PNG/GIF)
✅ Image serving (public, CORS enabled)
✅ Auto-creates bouquets when image uploaded
✅ Status reporting (water level, temp, humidity)
✅ Heartbeat simulation

## Bouquet Image Flow

```
Vendor/Admin Upload Image
         ↓
Mock Vase Service Stores Image
         ↓
Bouquet Entity Created/Updated
  (with imageUrl = /api/mock-vase/image/{vaseId})
         ↓
Customer Portal Queries Available Bouquets
         ↓
Bouquets Include Image URLs
         ↓
Map Markers Show Bouquet Images
         ↓
Customer Sees Beautiful Bouquet Photos!
```

## Testing the Complete Workflow

### Step-by-Step Test:

1. **Start All Services**:
   ```powershell
   # API (Docker)
   docker-compose up -d
   
   # Admin Portal
   cd src\portals\FlowerShop.AdminPortal
   dotnet run --launch-profile http
   
   # Vendor Portal  
   cd src\portals\FlowerShop.VendorPortal
   dotnet run --launch-profile http
   
   # Customer Portal
   cd src\portals\FlowerShop.CustomerApp
   dotnet run --launch-profile http
   ```

2. **Upload Bouquet Images** (Vendor Portal):
   - Login to http://localhost:5002
   - Go to "My Vases"
   - Click eye icon on ESP32-234567
   - Upload a beautiful bouquet photo
   - Repeat for ESP32-ASDFGH

3. **Verify Bouquets Created**:
   - Each upload auto-creates a bouquet
   - Rose Garden now has 2 bouquets (1 per vase)

4. **View on Customer Portal**:
   - Go to http://localhost:5003
   - See 2 round bouquet image markers on map
   - Click each to see full image and details

## Important Notes

### Image Persistence
- In this mock/testing path, images are stored **in-memory** (ConcurrentDictionary) and **cleared on API restart**
- **Production path**: bouquet photos upload to `POST /api/v1/bouquets/{id}/photo` and are stored durably in **Cloudflare R2** (served from `img.findmyflowers.pl`) via `IFileStorageService`; local disk in dev

### Vendor Location
- All vases inherit vendor's location
- Vases don't have separate addresses
- Makes sense: vase is physically at shop/home

### Authentication
- Mock vase service endpoints are **AllowAnonymous**
- Production: Implement device-specific tokens
- Vendors can upload without platform admin role

### Cache Busting
- Image URLs include `?t={timestamp}` 
- Ensures fresh loads after upload
- Prevents stale 404 cached responses

## Current State: Rose Garden

**Vendor**: Rose Garden (Traditional Shop)
**Location**: Geodetów 43, Józefosław (52.092345, 21.045100)
**Vases**: 2
  - ESP32-234567
  - ESP32-ASDFGH

**Expected Behavior**:
- Upload image to each vase
- 2 bouquets auto-created
- Customer sees 2 round image markers on map at Rose Garden's location
- Clicking markers shows uploaded bouquet photos

## Success Criteria ✅

- ✅ Rose Garden has 2 vases
- ✅ Each vase can have a bouquet image
- ✅ Images transmitted to Customer Portal
- ✅ Customer sees bouquets on map with photos
- ✅ Vendor Portal shows current bouquet thumbnails
- ✅ All location data correct

