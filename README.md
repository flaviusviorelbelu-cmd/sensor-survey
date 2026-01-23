# Sensor Survey - Satellite & Sensor Management System

**Version:** v5.6.9 (Stable - All Functions Working) ✅  
**Last Updated:** January 23, 2026  
**Location:** Den Haag, Netherlands  
**Repository:** [GitHub - sensor-survey](https://github.com/flaviusviorelbelu-cmd/sensor-survey)

---

## 📋 Table of Contents

1. [Executive Summary](#executive-summary)
2. [Project Status](#project-status)
3. [Architecture Overview](#architecture-overview)
4. [Data Model](#data-model)
5. [Features & Functions](#features--functions)
6. [Technical Details](#technical-details)
7. [Deployment & Integration](#deployment--integration)
8. [Troubleshooting](#troubleshooting)
9. [Version History](#version-history)

---

## Executive Summary

This project is a **web-based management system** for tracking satellite and sensor relationships in a SharePoint 2019 on-premises environment. It handles a **many-to-many relationship** where:

- One **Satellite** can have multiple **Sensors**
- One **Sensor** can be on multiple **Satellites**

**Current Status:** All functions fully operational and tested  
**Technology Stack:** Pure vanilla JavaScript, no external dependencies  
**Environment:** SharePoint 2019 (orionsps)

---

## Project Status

### ✅ Current Version: v5.6.9_FIXED

| Function | Status | Notes |
|----------|--------|-------|
| Export CSV | ✅ WORKING | Downloads all satellites and sensor data |
| Import CSV | ✅ WORKING | Uploads data from CSV file |
| Add Satellite | ✅ WORKING | Create new satellite with sensor relationships |
| Edit Satellite | ✅ WORKING | Modify satellite and update sensor links |
| Delete Satellite | ✅ WORKING | Remove satellite and cascade delete junctions |

**Development Timeline:**
- v5.4.1 - Initial version with broken Add/Edit
- v5.6.x - Iterative improvements and fixes
- v5.6.9 - Production stable (all functions working)

---

## Architecture Overview

### Two-List + Junction Structure

This is a normalized Many-to-Many relationship system using **three SharePoint lists**:

```
┌──────────────────────┐        ┌──────────────────────┐
│  Satellite_Fixed     │        │      Sensor          │
│  (Parent List)       │        │   (Parent List)      │
├──────────────────────┤        ├──────────────────────┤
│ ID (PK)              │        │ ID (PK)              │
│ Title                │        │ Title                │
│ norad_id (Unique)    │        │ sensor_id (Unique)   │
│ cospar_id            │        │ sensor_type          │
│ status               │        │ N_of_Bands           │
│ mission_type         │        │ Resolution_Min/Max   │
│ orbit_type           │        │ Swath_Min/Max        │
│ launch_date          │        │ ...                  │
│ ...                  │        │                      │
└──────────┬───────────┘        └──────────┬───────────┘
           │                               │
           │     1:N           N:1         │
           │      └─────────────┬──────────┘
           │                    │
           └────────┬───────────┘
                    │
          ┌─────────▼────────────┐
          │ Satellite_Sensor     │
          │   (Junction List)    │
          ├──────────────────────┤
          │ ID (PK)              │
          │ junction_id (Unique) │
          │ norad_id (FK)        │
          │ sensor_id (FK)       │
          │ Mission_cleaned      │
          └──────────────────────┘
```

---

## Data Model

### List 1: Satellite_Fixed (Parent)

**API Endpoint:** `http://orionsps/s/1/classic/_api/web/lists/getbytitle('Satellite_Fixed')/items`  
**Data File:** `Satellite_Fixed.txt`

| Field | Type | Description |
|-------|------|-------------|
| ID | System | Auto-generated |
| Title | Text | Satellite name (e.g., "NUSAT-3") |
| norad_id | Number | NORAD catalog number (PRIMARY KEY) |
| cospar_id | Text | COSPAR ID |
| status | Text | Operational status |
| constellation_id | Number | Reference to constellation |
| mission_type | Text | e.g., "Earth Observation" |
| detailed_purpose | Text | e.g., "Optical Imaging" |
| launch_site | Text | Launch location |
| launch_vehicle | Text | Launch rocket type |
| launch_date | DateTime | ISO 8601 format |
| expected_lifetime | Text | Expected lifetime |
| decay_date | DateTime | Predicted decay date |
| orbit_type | Text | LEO, GEO, MEO, etc. |
| comment | Text | Additional notes |
| alternate_name | Text | Alternative designation |
| sources | Text | Data sources/references |
| attachments | Text | Related file references |

### List 2: Sensor (Parent)

**API Endpoint:** `http://orionsps/s/1/classic/_api/web/lists/getbytitle('Sensor')/items`  
**Data File:** `Sensor.txt`

| Field | Type | Description |
|-------|------|-------------|
| ID | System | Auto-generated |
| Title | Text | Sensor name (e.g., "SAR-C (GF-3)") |
| sensor_id | Number | Unique sensor identifier (PRIMARY KEY) |
| sensor_type | Text | Type of sensor |
| description | Text | Sensor description |
| N_of_Bands | Number | Number of spectral bands |
| band_types | Text | Band types/wavelengths |
| Mission | Text | Mission name |
| Resolution_Min | Number | Minimum resolution (m) |
| Resolution_Max | Number | Maximum resolution (m) |
| Swath_Min | Number | Minimum swath (km) |
| Swath_Max | Number | Maximum swath (km) |

### List 3: Satellite_Sensor (Junction)

**API Endpoint:** `http://orionsps/s/1/classic/_api/web/lists/getbytitle('Satellite_Sensor')/items`

| Field | Type | Description |
|-------|------|-------------|
| ID | System | Auto-generated |
| junction_id | Number | Unique relationship identifier |
| norad_id | Number | Reference to Satellite_Fixed |
| sensor_id | Number | Reference to Sensor |
| Mission_cleaned | Text | Mission metadata |

---

## Features & Functions

### 1. Export CSV ✅

**Purpose:** Download all satellites and sensor relationships to CSV  
**Trigger:** "Export CSV" button  
**Output:** CSV file with headers and all data

**Process:**
```
1. Fetch all items from Satellite_Fixed list
2. Map to CSV format with column headers
3. Create downloadable file
4. Trigger browser download
```

### 2. Import CSV ✅

**Purpose:** Upload satellite data from CSV file  
**Trigger:** File input + "Import" button  

**Process:**
```
1. Parse CSV file (validate format)
2. Create items in Satellite_Fixed list
3. Create junction records in Satellite_Sensor list
4. Verify sensor relationships exist
5. Refresh UI with new data
```

**Expected CSV Format:**
```
name_satellite,norad_id,cospar_id,status,mission_type,orbit_type,launch_date,sensor_ids
NUSAT-3,42760,,Active,Earth Observation,LEO,2017-06-15,"1,2"
NUSAT-25,52171,,Active,Earth Observation,LEO,2022-04-01,"3"
```

### 3. Add Satellite ✅

**Purpose:** Create new satellite record with related sensors  
**Trigger:** Form submission or "Add Satellite" button  

**Required Fields:**
- name_satellite (Text)
- norad_id (Number - must be unique)
- mission_type (Text)
- orbit_type (Text)
- Sensor selection (multi-select)

**Process:**
```
1. Validate required fields
2. Check for duplicate norad_id
3. POST item to Satellite_Fixed list
4. Get returned item ID
5. For each selected sensor:
   - POST junction record to Satellite_Sensor list
6. Refresh UI
```

### 4. Edit Satellite ✅

**Purpose:** Modify existing satellite record and sensor relationships  
**Trigger:** Edit button on existing record  

**Update Fields:** All Satellite_Fixed fields + sensor relationships  
**HTTP Method:** MERGE with `If-Match: *` header

**Process:**
```
1. Load current satellite data
2. Allow user to modify fields
3. Allow user to add/remove sensors
4. MERGE updated data to Satellite_Fixed
5. Delete old junction records
6. Create new junction records for updated sensors
7. Refresh UI
```

### 5. Delete Satellite ✅

**Purpose:** Remove satellite and all related junction records  
**Trigger:** Delete button + confirmation dialog  

**Cascade Process:**
```
1. Confirm user action
2. Find all Satellite_Sensor records with this satellite's norad_id
3. DELETE each junction record
4. DELETE the satellite record from Satellite_Fixed
5. Verify Sensor list remains intact (not deleted)
6. Refresh UI
```

**Important:** Sensor records are NOT deleted, only the junctions are removed.

---

## Technical Details

### REST API Configuration

**Authentication:** Forms-based (SharePoint 2019 default)  
**Protocol:** HTTP/HTTPS JSON

**Required Headers for Write Operations:**
```javascript
{
    'Accept': 'application/json',
    'Content-Type': 'application/json',
    'X-RequestDigest': document.getElementById('__REQUESTDIGEST').value,
    'If-Match': '*' // For MERGE operations
}
```

### Request Digest

- **Automatically provided** by SharePoint on page load
- **Element ID:** `__REQUESTDIGEST`
- **Validity:** ~30 minutes
- **If expired:** Page refresh or re-login required

### Data Type Conversions

| Excel/Input | SharePoint Type | JavaScript Conversion |
|-------------|-----------------|----------------------|
| String | Text | Direct assignment |
| Number | Number | `parseInt()` or `parseFloat()` |
| Date | DateTime | `new Date().toISOString()` |
| Dropdown | Choice | String value |
| List Item | Lookup | Use `FieldNameId` with numeric ID |

### Many-to-Many Junction Logic

```javascript
// Pattern for creating relationships:
For each satellite being added/edited:
  1. Get satellite's norad_id (unique identifier)
  2. For each selected sensor:
     a. Get sensor's ID from Sensor list
     b. Create record in Satellite_Sensor:
        {
          '__metadata': { type: 'SP.ListItem' },
          'norad_id': satellite.norad_id,
          'sensor_id': sensor.sensor_id,
          'junction_id': generateUniqueId()
        }
```

### Error Handling

**Current Implementation (v5.6.9):**
- Try-catch blocks on all API calls
- Console logging for debugging
- User alerts for operation results
- Response status validation (200-299)
- Rollback on cascade failure

**Common Errors:**
1. **400 Bad Request** → Invalid payload (check field names/types)
2. **401 Unauthorized** → Auth expired (refresh page/re-login)
3. **403 Forbidden** → No permissions (check user permissions)
4. **404 Not Found** → List doesn't exist (verify list name spelling)
5. **409 Conflict** → Item locked (wait and retry)

---

## Deployment & Integration

### How to Use the Script

**Step 1: Obtain the Script**
- Download `script_v5.6.9_FIXED.txt` from this repository
- Rename to `.html` if using standalone
- Keep as `.txt` if embedding in SharePoint

**Step 2: Deploy Location**
- Option A: SharePoint Script Editor web part
- Option B: Content Editor web part
- Option C: Standalone HTML file in local machine

**Step 3: Authentication**
- Uses current user's credentials (automatic)
- Forms-based authentication (SharePoint 2019 default)

**Step 4: Permissions Required**
- **Contribute:** To add/edit items
- **Read:** To view/export items
- **Delete:** To remove satellites

### Page Integration Example

```html
<!DOCTYPE html>
<html>
<head>
    <title>Sensor Survey Management</title>
</head>
<body>
    <div id="sensorSurveyApp">
        <h1>Satellite & Sensor Management</h1>
        <div class="controls">
            <button onclick="exportCSV()">📥 Export CSV</button>
            <button onclick="importCSV()">📤 Import CSV</button>
            <button onclick="addSatellite()">➕ Add Satellite</button>
            <button onclick="editSatellite()">✏️ Edit Satellite</button>
            <button onclick="deleteSatellite()">🗑️ Delete Satellite</button>
        </div>
        <div id="dataDisplay"></div>
    </div>

    <!-- Request Digest provided by SharePoint -->
    <input type="hidden" id="__REQUESTDIGEST" value="" />

    <script>
        // Embed script_v5.6.9_FIXED.txt content here
    </script>
</body>
</html>
```

### List Creation in SharePoint

**Create Satellite_Fixed List:**
1. SharePoint Site → New → List
2. Name: "Satellite_Fixed"
3. Add columns for: norad_id, cospar_id, status, mission_type, etc.
4. Set norad_id as unique (optional, recommended)

**Create Sensor List:**
1. Name: "Sensor"
2. Add columns for: sensor_id, sensor_type, N_of_Bands, Resolution_Min/Max, etc.

**Create Satellite_Sensor List:**
1. Name: "Satellite_Sensor"
2. Columns:
   - junction_id (Number)
   - norad_id (Number)
   - sensor_id (Number)
   - Mission_cleaned (Text)

---

## Troubleshooting

### Add/Edit Satellite Not Working

**Step 1: Verify Request Digest**
```javascript
console.log(document.getElementById('__REQUESTDIGEST').value);
// Should output 40+ character string, not empty/null
```

**Step 2: Check API Endpoint**
```javascript
fetch("/s/1/classic/_api/web/lists/getbytitle('Satellite_Fixed')/items?$top=1", {
    headers: { 'Accept': 'application/json' }
})
.then(r => r.json())
.then(data => console.log(data));
```

**Step 3: Check Response Details**
1. Open browser DevTools (F12)
2. Go to Network tab
3. Attempt failed operation
4. Click the request
5. Check Response tab for error message

### Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| "400 Bad Request" | Invalid payload | Check field names match exactly (case-sensitive) |
| "401 Unauthorized" | Auth expired | Refresh page or re-login |
| "403 Forbidden" | No permissions | Check user has "Contribute" on list |
| "404 Not Found" | List doesn't exist | Verify list name spelling (Satellite_Fixed exactly) |
| Data not saving | Lookup field error | Use `FieldNameId` syntax with numeric ID |
| Delete not working | Junctions not found | Verify Satellite_Sensor list has correct records |

### Data Not Persisting

- Check that SharePoint context is detected
- Verify user has appropriate list permissions
- Check if in standalone mode (use Export/Import for backup)
- Verify no JavaScript errors in console

### Performance Issues

- If list has 100K+ records, consider pagination
- Consider archiving old/inactive satellites
- Use filtering in CAML queries for faster retrieval

---

## Version History

| Version | Date | Status | Major Changes |
|---------|------|--------|---------------|
| v5.6.9 | Jan 23, 2026 | ✅ STABLE | All functions tested and working |
| v5.6.3 | Jan 16, 2026 | ✅ STABLE | Initial stable release |
| v5.4.1 | Jan 16, 2026 | ⚠️ BROKEN | Add/Edit functions had issues |
| v5.x | Earlier | 🔧 DEV | Various iterations |

---

## File Structure

```
sensor-survey/
├── README.md                        # This file
├── script_v5.6.9_FIXED.txt         # Current stable version (ALL FUNCTIONS WORKING)
├── Satellite_Fixed.txt             # Raw API export from list
├── Sensor.txt                      # Raw API export from list
├── NU_DataSource_modifiedFrom_ISR_data.xlsx  # Source Excel data
└── [archived versions]
```

---

## Dependencies

- **SharePoint 2019** on-premises
- **Modern browser** with Fetch API support (Chrome, Edge, Firefox)
- **JavaScript ES6+** features:
  - Arrow functions
  - Template literals
  - Array methods (map, filter, forEach)
  - Async/await

**No external libraries required** - Pure vanilla JavaScript

---

## Next Conversation Quick Reference

When restarting conversation, reference:

- [ ] **Project:** Sensor Survey (SharePoint 2019)
- [ ] **Current Version:** v5.6.9 - All functions working
- [ ] **Architecture:** Two lists + junction (Many-to-Many)
- [ ] **Issue/Goal:** [Describe what you need]

---

## Related Resources

### Internal Files
- Excel source: `NU_DataSource_modifiedFrom_ISR_data.xlsx`
- Satellite data: `Satellite_Fixed.txt`
- Sensor data: `Sensor.txt`

### External References
- [SharePoint 2019 REST API](https://learn.microsoft.com/en-us/sharepoint/dev/sp-add-ins/working-with-lists-and-list-items-with-rest)
- [NORAD Satellite Database](https://www.space-track.org/)
- [Celestrak](https://celestrak.org/)
- [University of Twente Sensor Database](https://www.itc.nl/facilities/satellite-sensor-database/)

---

## Support & Maintenance

**Developer:** Flavius Viorel Belu  
**Organization:** NCIA (NATO Scientific Network) / ROMPRO Foundation  
**Location:** Den Haag, Netherlands  
**Last Updated:** January 23, 2026

For issues or questions, refer to the Troubleshooting section above or check the GitHub issues.

---

**Document Status:** Ready for use | **All Functions:** ✅ Working
