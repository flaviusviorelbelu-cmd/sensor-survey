# Future Migration Guide: From 2-List to 3-List Junction Architecture

**Status:** Planning document | **Target Timeline:** When data exceeds 500 satellites  
**Effort Estimate:** 4-6 hours one-time | **Risk Level:** Low (reversible)

---

## When to Migrate

### Triggers for Migration

**Migrate WHEN:**
- ⚠️ Data reaches **500+ satellites**
- ⚠️ Frequent **add/remove sensor** operations
- ⚠️ **Data integrity** becomes critical
- ⚠️ Need to **rename sensors** frequently
- ⚠️ Performance **degrades with parsing overhead**
- ⚠️ Want **proper database normalization**

### Do NOT Migrate IF:
- ✓ Data volume stable at <100 satellites
- ✓ Relationships rarely change
- ✓ Script works reliably
- ✓ Data inconsistencies haven't occurred

---

## Architecture Change Overview

### Current (2-List)
```
Satellite_Fixed
├─ Sensor_Names: "SAR-C, VIIRS, AIS"
└─ Primary_Sensor: "SAR-C"
                ↕️ Manual sync
                ↕️ Text parsing
Sensor
├─ Satellite_Names: "NUSAT-3, NUSAT-25"
└─ Satellite_Count: 2
```

### Target (3-List with Junction)
```
Satellite_Fixed              Sensor
├─ NORAD_ID          ←──────→  Title
└─ (clean data)      Junction  (clean data)
                       List
                    ├─ Junction_ID
                    ├─ Satellite_NORAD_ID (FK)
                    └─ Sensor_Name (FK)
```

---

## Migration Timeline

### Phase 1: Preparation (30 mins)
```
1. Create backup of both lists
2. Create test environment
3. Verify data consistency
4. Document current state
```

### Phase 2: List Creation (30 mins)
```
1. Create new Satellite_Sensor (junction) list
2. Add columns:
   - Junction_ID (Number)
   - Satellite_NORAD_ID (Number)
   - Sensor_Name (Text)
   - Mission_Name (Text, optional)
3. Verify list creation
```

### Phase 3: Data Migration (60 mins)
```
1. Run PowerShell migration script
2. Verify junction table population
3. Validate all relationships migrated
4. Check for orphaned records
5. Verify counts match
```

### Phase 4: Script Update (60 mins)
```
1. Update v5.6.9 script logic:
   - Change add/remove to use junction
   - Update delete cascade logic
   - Simplify parsing (no more text split/join)
2. Test all CRUD operations
3. Verify sync with 3 lists
```

### Phase 5: Cleanup (30 mins)
```
1. Remove Sensor_Names from Satellite_Fixed
2. Remove Primary_Sensor from Satellite_Fixed
3. Remove Satellite_Names from Sensor
4. Remove Satellite_Count from Sensor
5. Archive old script version
6. Update README documentation
```

**Total Effort:** ~4 hours

---

## Step-by-Step Migration

### Step 1: Create Backup

```powershell
# PowerShell backup script
$date = Get-Date -Format "yyyy-MM-dd_HHmm"
$siteUrl = "http://orionsps/s/1/classic"

Connect-PnPOnline -Url $siteUrl

# Export current lists
$satellites = Get-PnPListItem -List "Satellite_Fixed" -PageSize 5000
$sensors = Get-PnPListItem -List "Sensor" -PageSize 5000

# Save to CSV
$satellites | Select-Object NORAD_ID, Title, Sensor_Names, Primary_Sensor | 
    Export-Csv -Path "backup_satellites_$date.csv" -NoTypeInformation
    
$sensors | Select-Object ID, Title, Satellite_Names, Satellite_Count | 
    Export-Csv -Path "backup_sensors_$date.csv" -NoTypeInformation

Write-Host "Backup complete: backup_satellites_$date.csv, backup_sensors_$date.csv"
```

### Step 2: Create Junction List

**In SharePoint UI:**
1. Site → New → List
2. Name: "Satellite_Sensor"
3. Add columns:
   - Junction_ID (Number, Required)
   - Satellite_NORAD_ID (Number, Required)
   - Sensor_Name (Text, Required)
   - Mission_Name (Text, Optional)

**Or via PowerShell:**

```powershell
$listName = "Satellite_Sensor"

# Create list
New-PnPList -Title $listName -Template GenericList

# Add columns
Add-PnPField -List $listName -DisplayName "Junction_ID" -InternalName "Junction_ID" -Type Number -Required
Add-PnPField -List $listName -DisplayName "Satellite_NORAD_ID" -InternalName "Satellite_NORAD_ID" -Type Number -Required
Add-PnPField -List $listName -DisplayName "Sensor_Name" -InternalName "Sensor_Name" -Type Text -Required
Add-PnPField -List $listName -DisplayName "Mission_Name" -InternalName "Mission_Name" -Type Text

Write-Host "Junction list created: $listName"
```

### Step 3: Run Migration Script

```powershell
function Migrate-To-JunctionModel {
    param(
        [string]$SiteUrl = "http://orionsps/s/1/classic"
    )
    
    Connect-PnPOnline -Url $SiteUrl
    
    # Get all data
    $satellites = Get-PnPListItem -List "Satellite_Fixed" -PageSize 5000
    $sensors = Get-PnPListItem -List "Sensor" -PageSize 5000
    $junctionList = "Satellite_Sensor"
    
    $junctionId = 1
    $migratedCount = 0
    
    # For each satellite
    foreach ($sat in $satellites) {
        $sensorNames = $sat.FieldValues["Sensor_Names"] -split "," | 
            ForEach-Object { $_.Trim() } |
            Where-Object { $_.Length -gt 0 }
        
        # For each sensor linked to this satellite
        foreach ($sensorName in $sensorNames) {
            $sensor = $sensors | Where-Object { $_.Title -eq $sensorName }
            
            if ($sensor) {
                # Create junction record
                Add-PnPListItem -List $junctionList -Values @{
                    "Junction_ID" = $junctionId
                    "Satellite_NORAD_ID" = $sat.FieldValues["NORAD_ID"]
                    "Sensor_Name" = $sensorName
                    "Mission_Name" = $sat.FieldValues["Mission_Type"]
                } | Out-Null
                
                $junctionId++
                $migratedCount++
                
                Write-Progress -Activity "Migrating relationships" `
                    -Status "Satellite: $($sat.Title), Sensor: $sensorName" `
                    -PercentComplete ([math]::Min(($migratedCount/$satellites.Count*100),99))
            }
            else {
                Write-Warning "Sensor not found: $sensorName (referenced by $($sat.Title))"
            }
        }
    }
    
    Write-Host "Migration complete: $migratedCount junction records created"
    return $migratedCount
}

# Run migration
$result = Migrate-To-JunctionModel
Write-Host "Created $result relationship records"
```

### Step 4: Verify Migration

```powershell
function Verify-Migration {
    param([string]$SiteUrl = "http://orionsps/s/1/classic")
    
    Connect-PnPOnline -Url $SiteUrl
    
    $satellites = Get-PnPListItem -List "Satellite_Fixed" -PageSize 5000
    $junctions = Get-PnPListItem -List "Satellite_Sensor" -PageSize 5000
    $sensors = Get-PnPListItem -List "Sensor" -PageSize 5000
    
    $errors = @()
    
    # Check each satellite has junctions
    foreach ($sat in $satellites) {
        $satJunctions = $junctions | Where-Object { 
            $_.FieldValues["Satellite_NORAD_ID"] -eq $sat.FieldValues["NORAD_ID"] 
        }
        
        $oldSensorCount = ($sat.FieldValues["Sensor_Names"] -split ",").Count
        $newSensorCount = $satJunctions.Count
        
        if ($oldSensorCount -ne $newSensorCount) {
            $errors += "Satellite $($sat.Title): Expected $oldSensorCount sensors, found $newSensorCount in junction"
        }
    }
    
    # Check each sensor has junctions
    foreach ($sensor in $sensors) {
        $sensorJunctions = $junctions | Where-Object { 
            $_.FieldValues["Sensor_Name"] -eq $sensor.Title 
        }
        
        if ($sensor.FieldValues["Satellite_Count"] -ne $sensorJunctions.Count) {
            $errors += "Sensor $($sensor.Title): Count is $($sensor.FieldValues['Satellite_Count']) but found $($sensorJunctions.Count) in junction"
        }
    }
    
    if ($errors.Count -eq 0) {
        Write-Host "✓ Verification passed! All relationships migrated correctly."
        return $true
    }
    else {
        Write-Host "✗ Verification failed. Errors:"
        $errors | ForEach-Object { Write-Host "  - $_" }
        return $false
    }
}

$verified = Verify-Migration
if (-not $verified) {
    Write-Error "Migration verification failed. Do NOT proceed with cleanup."
    exit
}
```

### Step 5: Update Script Logic

**Before (Text-Based):**
```javascript
// Add sensor to satellite
async function addSensorToSatellite(satelliteId, sensorName) {
    const sat = await getSatellite(satelliteId);
    const sensors = sat.Sensor_Names.split(',').map(s => s.trim());
    sensors.push(sensorName);
    const updated = sensors.join(', ');
    
    // Update satellite
    await updateSatellite(satelliteId, { Sensor_Names: updated });
    
    // Update sensor (reverse reference)
    const sensor = await getSensorByName(sensorName);
    const satellites = sensor.Satellite_Names.split(',').map(s => s.trim());
    satellites.push(sat.Title);
    await updateSensor(sensor.ID, { 
        Satellite_Names: satellites.join(', '),
        Satellite_Count: satellites.length
    });
}
```

**After (Junction-Based):**
```javascript
// Add sensor to satellite - MUCH SIMPLER!
async function addSensorToSatellite(satelliteId, sensorName) {
    const sat = await getSatellite(satelliteId);
    
    // Just insert one junction record
    await fetch(
        "/s/1/classic/_api/web/lists/getbytitle('Satellite_Sensor')/items",
        {
            method: 'POST',
            headers: getHeaders(),
            body: JSON.stringify({
                '__metadata': { type: 'SP.ListItem' },
                'Junction_ID': generateId(),
                'Satellite_NORAD_ID': sat.NORAD_ID,
                'Sensor_Name': sensorName,
                'Mission_Name': sat.Mission_Type
            })
        }
    );
}
```

### Step 6: Test CRUD Operations

```javascript
// Test all operations with new schema
async function testMigration() {
    console.log("Testing CRUD with new junction table...");
    
    // 1. Get satellites
    const sats = await getSatellites();
    console.log(`✓ Read ${sats.length} satellites`);
    
    // 2. Get sensors via junction
    const sensors = await getJunctionRelationships();
    console.log(`✓ Read ${sensors.length} sensor relationships via junction`);
    
    // 3. Add new satellite with sensor
    const newSat = await addSatellite({
        Title: "TEST-SAT",
        NORAD_ID: 99999
    });
    console.log("✓ Added satellite");
    
    // 4. Add sensor relationship via junction
    await addSensorToSatellite(newSat.ID, "VIIRS");
    console.log("✓ Added sensor via junction");
    
    // 5. Verify via query
    const verify = await getJunctionQuery(
        `NORAD_ID eq ${newSat.NORAD_ID}`
    );
    console.log(`✓ Verified: Satellite has ${verify.length} sensor(s)`);
    
    // 6. Delete satellite
    await deleteSatelliteAndJunctions(newSat.ID);
    console.log("✓ Deleted satellite and junctions");
    
    // 7. Verify deletion
    const afterDelete = await getJunctionQuery(
        `NORAD_ID eq ${newSat.NORAD_ID}`
    );
    if (afterDelete.length === 0) {
        console.log("✓ All tests passed!");
        return true;
    } else {
        console.error("✗ Junction records not deleted!");
        return false;
    }
}

await testMigration();
```

### Step 7: Clean Up Old Fields

**⚠️ ONLY AFTER:**
- Migration script runs successfully
- Verification passes
- Testing confirms all CRUD works
- Backup saved

```powershell
function Cleanup-OldFields {
    param([string]$SiteUrl = "http://orionsps/s/1/classic")
    
    Connect-PnPOnline -Url $SiteUrl
    
    Write-Host "Removing old relationship fields..."
    
    # Remove from Satellite_Fixed
    Remove-PnPField -List "Satellite_Fixed" -Identity "Sensor_Names" -Force
    Remove-PnPField -List "Satellite_Fixed" -Identity "Primary_Sensor" -Force
    
    # Remove from Sensor
    Remove-PnPField -List "Sensor" -Identity "Satellite_Names" -Force
    Remove-PnPField -List "Sensor" -Identity "Satellite_Count" -Force
    
    Write-Host "✓ Cleanup complete. Old fields removed."
}

# Warning prompt
Write-Host "This will remove the old text-based relationship fields."
Write-Host "Make sure you have:"
Write-Host "  1. Run migration script successfully"
Write-Host "  2. Verified all relationships in junction table"
Write-Host "  3. Tested new script logic"
Write-Host "  4. Saved backup"
$confirm = Read-Host "Continue with cleanup? (type 'yes' to confirm)"

if ($confirm -eq "yes") {
    Cleanup-OldFields
} else {
    Write-Host "Cleanup cancelled."
}
```

---

## Rollback Procedure

**If migration fails:**

```powershell
function Rollback-Migration {
    param([string]$BackupDate = "2026-01-23_1000")
    
    Write-Host "Rolling back to backup from $BackupDate"
    
    # Delete new junction table
    Remove-PnPList -Identity "Satellite_Sensor" -Force
    
    # Restore from backup
    # Note: Manual restore from CSV export
    
    Write-Host "Rollback complete. Restore data from CSV backups if needed."
}
```

---

## Benefits After Migration

### ✅ Data Integrity
- Referential integrity enforced
- No orphaned records possible
- Automatic cascading

### ✅ Simplified Code
```javascript
// Before: 10 lines of text parsing
const sensors = sat.Sensor_Names.split(',').map(s => s.trim());
sensors.push(newSensor);
sat.Sensor_Names = sensors.join(', ');

// After: 1 INSERT
await addJunctionRecord(satId, sensorName);
```

### ✅ Scalability
- Handles 100K+ records
- No performance degradation
- Efficient queries

### ✅ Maintainability
- Rename sensors in one place
- No sync issues
- Automatic reverse references

### ✅ Flexibility
- Add metadata to relationships (Mission_Name, etc.)
- Support complex queries
- Easy analytics

---

## Decision Checklist

**Before starting migration:**

- [ ] Data volume exceeds 500 satellites
- [ ] Backup created and tested
- [ ] Test environment available
- [ ] Migration script reviewed
- [ ] CRUD tests prepared
- [ ] Team aware of downtime (if any)
- [ ] Rollback procedure understood
- [ ] Documentation ready for update

---

## Post-Migration Checklist

**After completing migration:**

- [ ] All CRUD operations tested
- [ ] No orphaned records
- [ ] Satellite_Count matches junction table
- [ ] Old fields removed
- [ ] README updated
- [ ] Script v6.0 deployed
- [ ] User training completed
- [ ] Backup retained for 30 days

---

**Document Status:** Planning phase | **Ready when needed**
