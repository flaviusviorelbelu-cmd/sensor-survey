# Sensor Survey - Satellite & Sensor Management System

**Version:** v5.6.9 (Stable - All Functions Working) ✅  
**Last Updated:** January 23, 2026  
**Location:** Den Haag, Netherlands  
**Repository:** [GitHub - sensor-survey](https://github.com/flaviusviorelbelu-cmd/sensor-survey)

---

## ⚠️ IMPORTANT: Current Architecture (Denormalized 2-List Approach)

**This project currently uses a 2-list denormalized structure with text-based relationships.**

Read the [Current Architecture](#current-architecture) section below before making changes to understand how data relationships are stored.

---

## 📋 Table of Contents

1. [Current Architecture](#current-architecture)
2. [Project Status](#project-status)
3. [Features & Functions](#features--functions)
4. [Technical Details](#technical-details)
5. [How Relationships Work](#how-relationships-work)
6. [Deployment & Integration](#deployment--integration)
7. [Troubleshooting](#troubleshooting)
8. [Future Roadmap](#future-roadmap)
9. [Version History](#version-history)

---

## Current Architecture

### Overview: 2-List Denormalized Model

This project uses **two SharePoint 2019 lists** with embedded text-based relationship fields (comma-separated values) instead of a formal junction table.

```
┌──────────────────────────────────────┐        ┌─────────────────────────────────┐
│    Satellite_Fixed List              │        │      Sensor List                │
├──────────────────────────────────────┤        ├─────────────────────────────────┤
│ • ID (Primary Key)                   │        │ • ID (Primary Key)              │
│ • NORAD_ID (Unique Identifier)       │        │ • Title (Sensor Name)           │
│ • name_satellite                     │        │ • Sensor_Type                   │
│ • Mission_Type                       │        │ • Description                   │
│ • Orbit_Type                         │        │ • N_of_Bands                    │
│ • Status                             │        │ • Resolution_Min/Max_m          │
│ • Launch_Date                        │        │ • Swath_Min/Max_km              │
│ • ...other fields                    │        │ • Sources                       │
│ ┌────────────────────────────────────┤        │ ┌─────────────────────────────┐ │
│ │ Sensor_Names (TEXT)                │        │ │ Satellite_Names (TEXT)      │ │
│ │ "SAR-C, VIIRS, AIS"                │◄──────►│ │ "NUSAT-3, NUSAT-25"         │ │
│ │ (comma-separated list)             │        │ │ (comma-separated list)      │ │
│ │                                    │        │ │                             │ │
│ │ Primary_Sensor (TEXT)              │        │ │ Satellite_Count (NUMBER)    │ │
│ │ "SAR-C"                            │        │ │ 2                           │ │
│ └────────────────────────────────────┘        └─────────────────────────────┘ │
└──────────────────────────────────────┘        └─────────────────────────────────┘
```

### How Relationships Are Stored

**Relationships are NOT normalized.** Instead:
- **Satellite_Fixed.Sensor_Names** contains a comma-separated list of sensor names used on that satellite
- **Sensor.Satellite_Names** contains a comma-separated list of satellite names that use that sensor
- **Sensor.Satellite_Count** contains the count of satellites using that sensor

**Example Data:**
```
Satellite_Fixed List (ID=1):
├─ NORAD_ID: 42760
├─ name_satellite: "NUSAT-3"
├─ Sensor_Names: "SAR-C (GF-3), VIIRS, AIS-RX"
└─ Primary_Sensor: "SAR-C (GF-3)"

Sensor List (ID=3):
├─ Title: "SAR-C (GF-3)"
├─ Sensor_Type: "SAR"
├─ Satellite_Names: "NUSAT-3, NUSAT-25, NUSAT-26"
└─ Satellite_Count: 3
```

### Why Text-Based Relationships?

- ✅ **Simpler UI** - fewer lists to manage
- ✅ **Faster queries** - 2 lists instead of 3
- ✅ **Suitable for current data volume** (~100 satellites, ~30 sensors)
- ⚠️ **Trade-off:** Manual relationship sync between lists required

---

## Project Status

### ✅ Current Version: v5.6.9_FIXED

All CRUD functions are fully operational:

| Function | Status | Notes |
|----------|--------|-------|
| Export CSV | ✅ WORKING | Downloads all satellites with sensor data |
| Import CSV | ✅ WORKING | Uploads satellites from CSV |
| Add Satellite | ✅ WORKING | Create new satellite + sensor relationships |
| Edit Satellite | ✅ WORKING | Modify satellite + update sensor links |
| Delete Satellite | ✅ WORKING | Remove satellite + cascade delete from sensor lists |

**Development Timeline:**
- v5.4.1 - Initial version (had Add/Edit issues)
- v5.4.7 - Iterative improvements
- v5.6.x - Additional refinements
- v5.6.9 - **STABLE** (all functions working)

---

## Features & Functions

### 1. Export CSV ✅

**Purpose:** Download all satellites and sensor relationships to CSV file

**Process:**
1. Fetch all items from Satellite_Fixed list
2. Map data to CSV format
3. Include Sensor_Names in export
4. Create downloadable file

**Output Format:**
```csv
name_satellite,NORAD_ID,COSPAR_ID,Mission_Type,Status,Orbit_Type,Sensor_Names,Primary_Sensor
NUSAT-3,42760,,Earth Observation,Active,LEO,"SAR-C, VIIRS",SAR-C
NUSAT-25,52171,,Earth Observation,Active,LEO,"3MI, MODIS",3MI
```

### 2. Import CSV ✅

**Purpose:** Upload satellite data from CSV file

**Input Format Expected:**
```csv
name_satellite,NORAD_ID,Sensor_Names,Primary_Sensor
NUSAT-3,42760,"SAR-C, VIIRS",SAR-C
```

**Process:**
1. Parse CSV file
2. For each row:
   - Create item in Satellite_Fixed list
   - Parse Sensor_Names (comma-separated)
   - For each sensor:
     - Find or create Sensor in Sensor list
     - Update Satellite_Names in Sensor list
     - Update Satellite_Count in Sensor list

### 3. Add Satellite ✅

**Purpose:** Create new satellite record with sensor relationships

**Required Fields:**
- name_satellite (Text)
- NORAD_ID (Number - must be unique)
- Mission_Type (Text)
- Orbit_Type (Text)
- Sensor_Names (comma-separated sensor names)
- Primary_Sensor (main sensor name)

**Process:**
```
1. Validate NORAD_ID is unique
2. Create item in Satellite_Fixed:
   {
     Title: name_satellite,
     NORAD_ID: value,
     Sensor_Names: "Sensor1, Sensor2, ...",
     Primary_Sensor: "Sensor1"
   }
3. For each sensor in Sensor_Names:
   - Find Sensor in Sensor list (by name)
   - Parse Satellite_Names from that sensor
   - Add current satellite to that list
   - Increment Satellite_Count
   - Update Sensor item
4. Refresh UI
```

⚠️ **Important:** Script must parse comma-separated values correctly

### 4. Edit Satellite ✅

**Purpose:** Modify satellite record and update sensor relationships

**Update Process:**
```
1. Load current satellite data
2. User modifies fields (including Sensor_Names)
3. MERGE update to Satellite_Fixed
4. Parse old vs new Sensor_Names:
   - Find removed sensors → remove from their Satellite_Names
   - Find added sensors → add to their Satellite_Names
5. Update all affected Sensor records
6. Recalculate Satellite_Count for affected sensors
7. Refresh UI
```

**Complexity:** This function must handle:
- Parsing old sensor list
- Parsing new sensor list
- Finding differences
- Updating reverse references
- Recalculating counts

### 5. Delete Satellite ✅

**Purpose:** Remove satellite and update all sensor references

**Cascade Process:**
```
1. Confirm user action
2. Parse Sensor_Names from satellite
3. For each sensor:
   - Find Sensor in Sensor list
   - Remove satellite name from Satellite_Names
   - Decrement Satellite_Count
   - Update Sensor item
4. DELETE satellite from Satellite_Fixed
5. Verify no orphaned references
6. Refresh UI
```

---

## How Relationships Work

### Adding a Sensor to a Satellite

**Scenario:** You want to add "VIIRS" sensor to "NUSAT-3" (which currently has "SAR-C")

**Steps:**
```javascript
// In Satellite_Fixed (NUSAT-3):
1. Read: Sensor_Names = "SAR-C"
2. Parse: ["SAR-C"]
3. Add new: ["SAR-C", "VIIRS"]
4. Rejoin: "SAR-C, VIIRS"
5. Update Satellite_Fixed with new Sensor_Names

// In Sensor list:
6. Read Sensor "VIIRS":
   - Satellite_Names = "...other satellites..."
7. Parse existing satellites
8. Add "NUSAT-3"
9. Rejoin: "...other satellites..., NUSAT-3"
10. Increment Satellite_Count
11. Update Sensor record
```

**Risk Factors:**
- ⚠️ Typo in sensor name ("VIIRS" vs "Viirs") breaks link
- ⚠️ Script parsing error can corrupt data
- ⚠️ Concurrent edits cause sync issues

### Renaming a Sensor

**Scenario:** Rename "SAR-C" to "SAR-C-BAND"

**Current Process (Manual):**
```
1. Find all Satellites with "SAR-C" in Sensor_Names
2. Update each satellite's Sensor_Names text
3. Update Sensor.Title
4. Verify no broken references
⚠️ Error-prone, time-consuming
```

**Why This Is Risky:**
- If you miss a satellite, it will have orphaned reference
- No automatic validation to find broken links
- Requires manual cleanup

---

## Technical Details

### REST API Endpoints

**Satellite_Fixed List:**
```
http://orionsps/s/1/classic/_api/web/lists/getbytitle('Satellite_Fixed')/items
```

**Sensor List:**
```
http://orionsps/s/1/classic/_api/web/lists/getbytitle('Sensor')/items
```

### Required Headers

```javascript
{
    'Accept': 'application/json',
    'Content-Type': 'application/json',
    'X-RequestDigest': document.getElementById('__REQUESTDIGEST').value,
    'If-Match': '*' // For MERGE operations
}
```

### Data Type Conversions

| Field Type | JavaScript Type | Conversion |
|------------|-----------------|------------|
| Text (Sensor_Names) | String | Direct assignment |
| Number (NORAD_ID) | Number | `parseInt()` |
| DateTime (Launch_Date) | String | `new Date().toISOString()` |
| Choice (Orbit_Type) | String | Direct assignment |
| Note (Detailed_Purpose) | String | Direct assignment |

### Error Handling in Current Script

```javascript
// Try-catch on API calls
try {
    const response = await fetch(apiUrl, {
        method: 'POST',
        headers: requiredHeaders,
        body: JSON.stringify(payload)
    });
    
    if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    }
    
    return await response.json();
} catch (error) {
    console.error('API Error:', error);
    alert('Operation failed: ' + error.message);
}
```

---

## Deployment & Integration

### How to Use the Script

**Step 1: Obtain Script**
- Download `script_v5.6.9_FIXED.txt` from repository
- Or rename to `.html` for standalone use

**Step 2: Deploy to SharePoint**
- Option A: Script Editor web part
- Option B: Content Editor web part
- Option C: Standalone HTML file

**Step 3: Verify Lists Exist**
- List 1: "Satellite_Fixed" with NORAD_ID, Sensor_Names, Primary_Sensor fields
- List 2: "Sensor" with Satellite_Names, Satellite_Count fields

**Step 4: Test CRUD Operations**
```
1. Export CSV (verify format)
2. Add Satellite (test sensor parsing)
3. Edit Satellite (verify sync to Sensor list)
4. Delete Satellite (verify reverse reference cleanup)
```

### List Creation Steps

**Create Satellite_Fixed List:**
```
1. Site → New → List
2. Name: "Satellite_Fixed"
3. Add columns:
   - NORAD_ID (Number, Required)
   - COSPAR_ID (Text)
   - Mission_Type (Choice)
   - Orbit_Type (Choice)
   - Status (Choice)
   - Launch_Date (DateTime)
   - Detailed_Purpose (Note)
   - Comment (Note)
   - Sensor_Names (Note) ← KEY FIELD
   - Primary_Sensor (Text) ← KEY FIELD
   - Sources (URL)
```

**Create Sensor List:**
```
1. Site → New → List
2. Name: "Sensor"
3. Add columns:
   - Sensor_Type (Choice)
   - Description (Note)
   - N_of_Bands (Number)
   - Resolution_Min_m (Number)
   - Resolution_Max_m (Number)
   - Swath_Min_km (Number)
   - Swath_Max_km (Number)
   - Band_Types (Note)
   - Satellite_Names (Note) ← KEY FIELD
   - Satellite_Count (Number) ← KEY FIELD
   - Sources (URL)
```

### Page Integration Example

```html
<!DOCTYPE html>
<html>
<head>
    <title>Sensor Survey - Satellite Management</title>
</head>
<body>
    <h1>Satellite & Sensor Management</h1>
    
    <div class="controls">
        <button onclick="exportCSV()">📥 Export CSV</button>
        <button onclick="importCSV()">📤 Import CSV</button>
        <button onclick="addSatellite()">➕ Add Satellite</button>
        <button onclick="editSatellite()">✏️ Edit Satellite</button>
        <button onclick="deleteSatellite()">🗑️ Delete Satellite</button>
    </div>
    
    <div id="dataDisplay"></div>
    
    <!-- SharePoint provides this automatically -->
    <input type="hidden" id="__REQUESTDIGEST" />
    
    <script>
        // Embed script_v5.6.9_FIXED.txt here
    </script>
</body>
</html>
```

---

## Troubleshooting

### Add/Edit Functions Not Working

**Step 1: Verify Request Digest**
```javascript
console.log(document.getElementById('__REQUESTDIGEST').value);
// Should output 40+ character string
```

**Step 2: Check API Endpoint**
```javascript
fetch("/s/1/classic/_api/web/lists/getbytitle('Satellite_Fixed')/items?$top=1", {
    headers: { 'Accept': 'application/json' }
})
.then(r => r.json())
.then(data => console.log(data));
```

**Step 3: Verify Sensor List Fields**
Check that Sensor list has:
- ✅ Satellite_Names (Multiple lines of text)
- ✅ Satellite_Count (Number)

**Step 4: Check Sensor Name Consistency**
- Ensure sensor names match exactly ("VIIRS" ≠ "Viirs")
- Check for extra spaces
- Verify comma formatting in Sensor_Names

### Data Sync Issues

**Problem:** Satellite_Names in Sensor list doesn't match satellites that reference it

**Cause:** Manual list edits or script parsing error

**Solution:**
1. Export data from both lists
2. Verify consistency
3. Run cleanup script (PowerShell)
4. Re-import if needed

### Orphaned References

**Problem:** Satellite references a sensor that no longer exists

**Cause:** Manual deletion of sensor without updating satellites

**Solution:**
```javascript
// Find orphaned sensors:
function findOrphans() {
    const sensors = getSensorList();
    const satellites = getSatelliteList();
    
    satellites.forEach(sat => {
        const sensorNames = sat.Sensor_Names.split(',').map(s => s.trim());
        sensorNames.forEach(sensorName => {
            if (!sensors.find(s => s.Title === sensorName)) {
                console.warn(`ORPHAN: ${sat.name_satellite} references missing sensor: ${sensorName}`);
            }
        });
    });
}
```

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| "400 Bad Request" | Invalid payload or field name | Check field internal names match exactly |
| "401 Unauthorized" | Auth token expired | Refresh page or re-login |
| "403 Forbidden" | No permissions | Check user has "Contribute" permission |
| "404 Not Found" | List doesn't exist | Verify list name spelling |
| Sensor_Names won't update | Parsing error in script | Check comma-separated format |
| Satellite_Count wrong | Sync issue between lists | Manually recalculate |

---

## Future Roadmap

### Short Term (Next 1-2 Months)
- ✅ Document current 2-list architecture (DONE)
- Add validation to prevent sensor name typos
- Add sync verification script
- Improve error messages

### Medium Term (3-6 Months)
- **Monitor data growth** - If >500 satellites, consider migration
- Add cleanup utilities
- Create backup/restore functionality

### Long Term (6+ Months)

#### ⭐ PLANNED: Migration to 3-List Junction Architecture

When you're ready for production-grade normalization:

**Create new "Satellite_Sensor" junction list:**
```
Columns:
- Junction_ID (Number) - Unique identifier
- Satellite_NORAD_ID (Number) - Foreign key to Satellite_Fixed
- Sensor_Name (Text) - Foreign key to Sensor
- Mission_Name (Text) - Optional metadata
```

**Benefits:**
- ✅ Proper database normalization
- ✅ Automatic data integrity (no sync issues)
- ✅ Simple add/remove operations
- ✅ Easy sensor renaming
- ✅ Scales to 100K+ records
- ✅ Prevents orphaned data

**Migration effort:** ~4 hours (one-time)
- 1 hour: Create junction list
- 2 hours: PowerShell migration script
- 1 hour: Update app logic

**Decision point:** Implement when:
- Data volume exceeds 500+ satellites
- Data integrity becomes critical
- You add/remove sensors frequently

---

## File Structure

```
sensor-survey/
├── README.md                              # This file
├── ARCHITECTURE-COMPARISON.md             # Current vs Proposed (3-list) comparison
├── ACTUAL-CURRENT-ARCHITECTURE.md         # Detailed explanation of 2-list design
├── script_v5.6.9_FIXED.txt               # Current stable version (ALL WORKING)
├── Satellite_Fixed_Columns.csv           # Current list schema export
├── Sensor_Columns.csv                    # Current list schema export
├── Satellite_Fixed.txt                   # Raw API export from list
├── Sensor.txt                            # Raw API export from list
├── NU_DataSource_modifiedFrom_ISR_data.xlsx  # Source data (Excel)
└── [archived versions]
```

---

## Dependencies

- **SharePoint 2019** on-premises (orionsps)
- **Modern browser** supporting:
  - Fetch API
  - ES6+ JavaScript (arrow functions, template literals, async/await)
- **Chrome, Edge, or Firefox** (latest versions)
- **No external libraries** - Pure vanilla JavaScript

---

## Data Management

### Regular Maintenance

**Weekly:**
- Export CSV backup
- Verify Satellite_Count accuracy

**Monthly:**
- Check for orphaned references
- Validate sensor names for consistency
- Review script logs

**Before Bulk Operations:**
1. Export current data
2. Store backup with timestamp
3. Test operation on small dataset first
4. Verify reverse references after completion

### Backup Strategy

```powershell
# PowerShell backup script
$date = Get-Date -Format "yyyy-MM-dd_HHmm"
$satellites = Export-SatelliteData
$sensors = Export-SensorData

$satellites | Export-Csv "backup_satellites_$date.csv" -NoTypeInformation
$sensors | Export-Csv "backup_sensors_$date.csv" -NoTypeInformation
```

---

## Support & Maintenance

**Developer:** Flavius Viorel Belu  
**Organization:** BKF IT 
**Location:** Den Haag, Netherlands  
**Last Updated:** January 23, 2026

### Documentation Files
- **README.md** (this file) - Overview and usage
- **ARCHITECTURE-COMPARISON.md** - Detailed architecture comparison
- **ACTUAL-CURRENT-ARCHITECTURE.md** - Current 2-list design rationale

### External References
- [SharePoint 2019 REST API](https://learn.microsoft.com/en-us/sharepoint/dev/sp-add-ins/working-with-lists-and-list-items-with-rest)
- [NORAD Satellite Database](https://www.space-track.org/)
- [Celestrak](https://celestrak.org/)
- [University of Twente Sensor Database](https://www.itc.nl/facilities/satellite-sensor-database/)

---

## Version History

| Version | Date | Status | Notes |
|---------|------|--------|-------|
| v5.6.9 | Jan 23, 2026 | ✅ STABLE | All functions working - 2-list architecture |
| v5.6.3 | Jan 16, 2026 | ✅ STABLE | Initial stable release |
| v5.4.1 | Jan 16, 2026 | ⚠️ BROKEN | Add/Edit issues identified |
| v5.x | Earlier | 🔧 DEVELOPMENT | Various iterations |

---

**Document Status:** Ready for use | **All Functions:** ✅ Working | **Architecture:** 2-List Denormalized
