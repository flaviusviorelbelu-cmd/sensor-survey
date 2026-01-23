# Current Architecture: 2-List Denormalized Model

**Date:** January 23, 2026  
**Version:** Describes v5.6.9 implementation

---

## Executive Summary

The Sensor Survey project currently uses a **2-list denormalized architecture** where:
- **Relationships are stored as comma-separated text** in list items
- **No junction/bridge table exists**
- Both lists contain references to each other (bidirectional links)
- The script (v5.6.9) handles parsing and keeping references in sync

This approach is suitable for current data volume (~100 satellites, ~30 sensors) but requires careful relationship management.

---

## List 1: Satellite_Fixed

**Purpose:** Store satellite information with embedded sensor references

**API Endpoint:** `http://orionsps/s/1/classic/_api/web/lists/getbytitle('Satellite_Fixed')/items`

### Core Columns

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| ID | Counter | Auto | SharePoint auto-generated ID |
| Title (name_satellite) | Text | Yes | Satellite name (e.g., "NUSAT-3") |
| NORAD_ID | Number | Yes | NORAD catalog number (unique) |
| COSPAR_ID | Text | No | COSPAR designation |
| Mission_Type | Choice | No | Earth Observation, Communications, etc. |
| Detailed_Purpose | Note | No | Detailed mission purpose |
| Status | Choice | No | Active, Inactive, Planned, etc. |
| Constellation_ID | Number | No | Reference to constellation |
| Launch_Site | Text | No | Launch location |
| Launch_Vehicle | Text | No | Launch rocket type |
| Launch_Date | DateTime | No | Launch date |
| Expected_Lifetime | Number | No | Years of expected operation |
| Decay_Date | DateTime | No | Predicted decay date |
| Orbit_Type | Choice | No | LEO, GEO, MEO, HEO, etc. |
| Alternate_Name | Text | No | Alternative designation |
| Comment | Note | No | Additional notes |
| Sources | URL | No | Data source reference |

### Relationship Columns (KEY FIELDS)

| Column | Type | Purpose | Format |
|--------|------|---------|--------|
| **Sensor_Names** | Note | List of sensors on this satellite | Comma-separated: "VIIRS, SAR-C, AIS" |
| **Primary_Sensor** | Text | Main/primary sensor | Single value: "VIIRS" |

### Example Data

```
ID: 1
Title: NUSAT-3
NORAD_ID: 42760
Mission_Type: Earth Observation
Orbit_Type: LEO
Launch_Date: 2017-06-15
Status: Active
Sensor_Names: "SAR-C (GF-3), VIIRS, AIS-RX"
Primary_Sensor: "SAR-C (GF-3)"
```

---

## List 2: Sensor

**Purpose:** Store sensor specifications with embedded satellite references

**API Endpoint:** `http://orionsps/s/1/classic/_api/web/lists/getbytitle('Sensor')/items`

### Core Columns

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| ID | Counter | Auto | SharePoint auto-generated ID |
| Title (name_satellite) | Text | Yes | Sensor name (e.g., "VIIRS") |
| Sensor_Type | Choice | No | SAR, Optical, Microwave, etc. |
| Description | Note | No | Sensor description |
| N_of_Bands | Number | No | Number of spectral bands |
| Band_Types | Note | No | List of band types/wavelengths |
| Resolution_Min_m | Number | No | Minimum resolution (meters) |
| Resolution_Max_m | Number | No | Maximum resolution (meters) |
| Swath_Min_km | Number | No | Minimum swath (km) |
| Swath_Max_km | Number | No | Maximum swath (km) |
| Other_Metrics | Note | No | Additional specifications |
| Sources | URL | No | Data source reference |

### Relationship Columns (KEY FIELDS)

| Column | Type | Purpose | Format |
|--------|------|---------|--------|
| **Satellite_Names** | Note | List of satellites using this sensor | Comma-separated: "NUSAT-3, NUSAT-25, NUSAT-26" |
| **Satellite_Count** | Number | Count of satellites using this sensor | Number: 3 |

### Example Data

```
ID: 3
Title: SAR-C (GF-3)
Sensor_Type: SAR
Description: C-SAR on GF-3
Resolution_Min_m: 5
Resolution_Max_m: 100
Swath_Min_km: 50
Swath_Max_km: 400
Satellite_Names: "NUSAT-3, NUSAT-25, NUSAT-26"
Satellite_Count: 3
```

---

## Relationship Model

### How Relationships Are Stored

**Satellites track their sensors:**
```
Satellite_Fixed.Sensor_Names = "Sensor1, Sensor2, Sensor3"
Satellite_Fixed.Primary_Sensor = "Sensor1"
```

**Sensors track their satellites:**
```
Sensor.Satellite_Names = "Satellite1, Satellite2, ..."
Sensor.Satellite_Count = N
```

### Bidirectional Sync Requirement

When a relationship changes, **BOTH lists must be updated**:

```
Add SAR-C to NUSAT-3:
  ┌─────────────────────────────────────────┐
  │ Satellite_Fixed (NUSAT-3)               │
  │ Sensor_Names: "..." → "..., SAR-C"    │
  └─────────────────────────────────────────┘
                    ↓ AND ↓
  ┌─────────────────────────────────────────┐
  │ Sensor (SAR-C)                          │
  │ Satellite_Names: "..." → "..., NUSAT-3"│
  │ Satellite_Count: N → N+1                │
  └─────────────────────────────────────────┘
```

### Risk: Unsync Data

If only one list is updated, data becomes inconsistent:

```
❌ BROKEN STATE:
Satellite_Fixed (NUSAT-3):
  Sensor_Names: "VIIRS, SAR-C, AIS"

Sensor (SAR-C):
  Satellite_Names: "NUSAT-25, NUSAT-26"  ← Missing NUSAT-3!
  Satellite_Count: 2

⚠️ Script will find inconsistency during add/edit
```

---

## Data Operations

### Add Sensor to Satellite

**Script Logic:**
```javascript
1. Get current Sensor_Names from satellite
2. Split by comma → array
3. Add new sensor name to array
4. Check sensor exists in Sensor list
5. Join array → comma-separated string
6. Update satellite's Sensor_Names field
7. Update satellite's Primary_Sensor if first sensor
8. Get Sensor item from Sensor list
9. Add satellite to Sensor's Satellite_Names
10. Increment Sensor's Satellite_Count
11. Update Sensor item
```

**Parsing Requirements:**
- Handle whitespace: "VIIRS, SAR-C" (comma + space)
- Normalize names: Ensure exact match with Sensor list
- Validate sensor exists before adding
- Prevent duplicates (don't add if already present)

### Remove Sensor from Satellite

**Script Logic:**
```javascript
1. Get current Sensor_Names from satellite
2. Split by comma → array
3. Remove sensor name from array
4. Join array → comma-separated string
5. If removed sensor was Primary_Sensor:
   - Set Primary_Sensor to first remaining sensor
6. Update satellite's Sensor_Names field
7. Get Sensor item from Sensor list
8. Remove satellite from Sensor's Satellite_Names
9. Decrement Sensor's Satellite_Count
10. Update Sensor item
```

### Edit Satellite (Change Sensors)

**Script Logic:**
```javascript
1. Get old Sensor_Names (before edit)
2. Get new Sensor_Names (after edit)
3. Split both → arrays
4. Find sensors that were removed
5. Find sensors that were added
6. For each removed sensor:
   - Remove satellite from Sensor.Satellite_Names
   - Decrement Sensor.Satellite_Count
   - Update Sensor item
7. For each added sensor:
   - Add satellite to Sensor.Satellite_Names
   - Increment Sensor.Satellite_Count
   - Update Sensor item
8. Update satellite's Sensor_Names and Primary_Sensor
```

### Delete Satellite

**Script Logic:**
```javascript
1. Get Sensor_Names from satellite
2. Split by comma → array
3. For each sensor in array:
   - Get Sensor item
   - Remove satellite from Sensor.Satellite_Names
   - Decrement Sensor.Satellite_Count
   - Update Sensor item
4. Delete satellite from Satellite_Fixed
```

**Important:** Sensor records are NOT deleted, only the reverse reference is removed.

---

## Parsing Specification

### Comma-Separated Format

**Standard Format:**
```
"Sensor1, Sensor2, Sensor3"
```

**Rules:**
- Delimiter: Comma (`","`)
- Separator: Comma followed by space (`", "`)
- No leading/trailing spaces in names
- No duplicate names

**Examples:**
```
✅ Valid:
"VIIRS, SAR-C, AIS-RX"
"SAR-C (GF-3), VIIRS, AIS"
"Single Sensor"

❌ Invalid:
"VIIRS,SAR-C" (missing space)
"VIIRS , SAR-C" (space before comma)
"VIIRS, SAR-C," (trailing comma)
" VIIRS, SAR-C" (leading space)
"VIIRS, SAR-C, VIIRS" (duplicate)
```

### Parsing Algorithm

```javascript
function parseSensorNames(text) {
    return text
        .split(',')
        .map(s => s.trim())
        .filter(s => s.length > 0)
        .filter((v, i, a) => a.indexOf(v) === i); // Remove duplicates
}

function formatSensorNames(array) {
    return array.map(s => s.trim()).join(', ');
}

// Usage:
const text = "VIIRS, SAR-C, AIS";
const array = parseSensorNames(text);
// Result: ["VIIRS", "SAR-C", "AIS"]

array.push("MODIS");
const updated = formatSensorNames(array);
// Result: "VIIRS, SAR-C, AIS, MODIS"
```

---

## Error Scenarios

### Scenario 1: Typo in Sensor Name

**Problem:**
```
Satellite_Names: "NUSAT-3, NUSAT-25"
Sensor_Names: "VIIRS, SAR-C" (correct)

User adds: "viirs" (lowercase) instead of "VIIRS"
Result: Satellite_Names: "NUSAT-3, viirs" (new entry!)
```

**Impact:** Creates orphaned reference - "viirs" doesn't exist in Sensor list

**Prevention:**
- Use dropdown selector instead of free text
- Implement case-insensitive comparison
- Add fuzzy matching for typo detection

### Scenario 2: Concurrent Edits

**Problem:**
```
User A edits NUSAT-3: Adds MODIS
User B edits NUSAT-3 (from older version): Adds AIS

Result: One change is lost (last write wins)
```

**Impact:** Data inconsistency

**Prevention:**
- Add conflict detection (compare before/after)
- Show warning if item changed since load
- Use version field check

### Scenario 3: Manual List Edit

**Problem:**
```
User manually edits Satellite_Fixed in SharePoint UI:
Changes Sensor_Names to "VIIRS, SAR-C, AIS"

But forgets to update Sensor list reverse references:
Viirs.Satellite_Names: "NUSAT-3"
SAR-C.Satellite_Names: "NUSAT-3"
AIS.Satellite_Names: "" (forgot to add!)
```

**Impact:** AIS.Satellite_Count stays at 0, but NUSAT-3 claims to have AIS

**Prevention:**
- Disable manual edits of Sensor_Names field
- Use form validation
- Add read-only toggle for relationship fields

---

## Maintenance Operations

### Verify Data Consistency

```javascript
function verifyConsistency() {
    const satellites = getSatelliteList();
    const sensors = getSensorList();
    const errors = [];
    
    // Check each satellite
    satellites.forEach(sat => {
        const sensorNames = parseSensorNames(sat.Sensor_Names);
        sensorNames.forEach(sensorName => {
            const sensor = sensors.find(s => s.Title === sensorName);
            if (!sensor) {
                errors.push(`Satellite ${sat.Title} references missing sensor: ${sensorName}`);
            } else if (!sensor.Satellite_Names.includes(sat.Title)) {
                errors.push(`Satellite ${sat.Title} has ${sensorName}, but ${sensorName} doesn't list ${sat.Title}`);
            }
        });
    });
    
    // Check Satellite_Count accuracy
    sensors.forEach(sensor => {
        const satelliteNames = parseSensorNames(sensor.Satellite_Names);
        if (satelliteNames.length !== sensor.Satellite_Count) {
            errors.push(`Sensor ${sensor.Title}: Count is ${sensor.Satellite_Count} but has ${satelliteNames.length} satellites`);
        }
    });
    
    return errors;
}
```

### Auto-Repair Inconsistencies

```powershell
# PowerShell script to fix sync issues
function Repair-SensorRelationships {
    param([string]$SiteUrl)
    
    # Connect to SharePoint
    Connect-PnPOnline -Url $SiteUrl
    
    # Get all satellites
    $satellites = Get-PnPListItem -List "Satellite_Fixed" -PageSize 5000
    $sensors = Get-PnPListItem -List "Sensor" -PageSize 5000
    
    foreach ($sat in $satellites) {
        $sensorNames = $sat.Sensor_Names -split "," | ForEach-Object { $_.Trim() }
        
        foreach ($sensorName in $sensorNames) {
            $sensor = $sensors | Where-Object { $_.Title -eq $sensorName }
            if ($sensor) {
                # Check if satellite is in Sensor.Satellite_Names
                if ($sensor.Satellite_Names -notlike "*$($sat.Title)*") {
                    # Add satellite to sensor
                    $newSatNames = "$($sensor.Satellite_Names), $($sat.Title)" | ForEach-Object { $_.Trim() }
                    Set-PnPListItem -List "Sensor" -Identity $sensor.ID `
                        -Values @{"Satellite_Names"=$newSatNames}
                }
            }
        }
    }
}
```

---

## Migration Path to 3-List Architecture

When ready to implement proper normalization, see [ARCHITECTURE-COMPARISON.md](./ARCHITECTURE-COMPARISON.md) for:
- Why 3-list junction approach is better
- Migration timeline and effort
- Script changes required
- Testing strategy

---

**Document Status:** Accurate for v5.6.9 | **Updated:** January 23, 2026
