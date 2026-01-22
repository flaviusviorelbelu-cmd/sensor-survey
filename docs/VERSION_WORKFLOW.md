# Version Management Workflow

## Overview

This document defines the systematic process for creating, testing, and releasing new versions of the Satellite Sensor Survey application.

---

## Version Naming Convention

```
script_v[MAJOR].[MINOR].[PATCH]_[STATUS].txt

Examples:
✅ script_v5.6.4_FIXED.txt     ✓ Correct
✅ script_v5.6.0_FINAL.txt     ✓ Correct
✅ script_v5.4.7_FIXED.txt     ✓ Correct (last working delete)
❌ script_v5.6.4.txt          ✗ Wrong (missing status)
❌ script_v5.6.4_work.txt      ✗ Wrong (status not FIXED/FINAL)
❌ script_latest.txt           ✗ Wrong (use version numbers)
```

### Status Labels

- **FIXED** = Tested and working, ready for use
- **FINAL** = Major release, significant refactoring or milestone
- **WIP** = Work In Progress (internal only, don't commit)
- **STABLE** = Old version confirmed working (optional suffix)

---

## Development Workflow

### Phase 1: Starting Development

**When:** You want to work on new features or fixes

```
1. Start with working version: script_v5.6.3_FIXED.txt
   ↓
2. Copy to new file: script_v5.6.4_WIP.txt  (internal only)
   ↓
3. Make changes
   ↓
4. Test thoroughly (see checklist below)
   ↓
5. If working → script_v5.6.4_FIXED.txt
   If broken → identify issue, try again
```

**Keep in mind:**
- Never overwrite existing FIXED versions
- Always create new version number
- Even small patches get a version bump (e.g., v5.6.3 → v5.6.4)

### Phase 2: Making Changes

**Best Practice:** Atomic changes

- Each version should address ONE category:
  - Bug fixes (add v0.0.1, e.g., v5.6.4)
  - New feature (add v0.1, e.g., v5.7.0)
  - Major refactor (add v1.0, e.g., v6.0.0)

**Example Evolution:**
```
v5.6.3_FIXED.txt
  ↓ Fix delete bug
v5.6.4_FIXED.txt
  ↓ Add new sensor filter
v5.7.0_FIXED.txt
  ↓ Complete UI redesign
v6.0.0_FINAL.txt
```

### Phase 3: Testing Checklist

**Before marking as _FIXED, test these:**

#### ✅ Core CRUD Operations

```javascript
// Test in console:

// 1. TEST ADD
console.log('=== TEST ADD ===' );
const newData = {
  Title: 'Test Sat ' + Date.now(),
  NORAD_ID: '99999',
  Status: 'Testing',
  Orbit_Type: 'LEO',
  Launch_Date: '2026-01-22'
};
formHandler.setFormData(newData);
formHandler.saveSatellite();
console.log('After add:', state.satellites.length);

// 2. TEST EDIT
console.log('=== TEST EDIT ===');
const toEdit = state.satellites[0];
if (toEdit) {
  editSatellite(toEdit.ID);
  // Change a field
  document.querySelector('#satellite-title').value = 'EDITED ' + toEdit.Title;
  saveSatellite();
  console.log('After edit, verify in table');
}

// 3. TEST DELETE (CRITICAL)
console.log('=== TEST DELETE ===');
const toDelete = state.satellites[state.satellites.length - 1];
if (toDelete) {
  const beforeCount = state.satellites.length;
  deleteSatellite(toDelete.ID);
  const afterCount = state.satellites.length;
  console.log('Before:', beforeCount, 'After:', afterCount, 'Deleted:', beforeCount - afterCount);
  if (beforeCount - afterCount === 1) {
    console.log('✅ DELETE WORKS');
  } else {
    console.log('❌ DELETE FAILED');
  }
}
```

#### ✅ Display & UI

```javascript
// Test in browser (visual checks)

// 1. Tables display correctly
  ☑ Satellite table shows all columns
  ☑ Sensor table shows all columns  
  ☑ No horizontal scrolling needed
  ☑ Data is visible (not cut off)

// 2. Modals work
  ☑ "Add Satellite" modal opens
  ☑ Form fields populate correctly
  ☑ "Close" button works
  ☑ "Cancel" button works
  ☑ "Save" button works
  ☑ Backdrop click closes modal

// 3. Detail panel
  ☑ Click satellite row → shows details
  ☑ Detail panel displays all fields
  ☑ Detail panel updates when row clicked
  ☑ Sensor details work similarly

// 4. Pagination (if v5.6.2+)
  ☑ Previous/Next buttons work
  ☑ Page indicator shows correct page
  ☑ 25 items per page
  ☑ No data loss when paging

// 5. Filtering & Search
  ☑ Search by satellite name works
  ☑ Filter by status works
  ☑ Results update on search
  ☑ Clear search resets display
```

#### ✅ Data Operations

```javascript
// Test data import/export

// 1. CSV EXPORT
  ☑ Click "Export" button
  ☑ CSV file downloads
  ☑ File has correct filename with data
  ☑ CSV is valid (can open in Excel)
  ☑ All satellites included

// 2. CSV IMPORT
  ☑ Click "Import" button
  ☑ File picker opens
  ☑ Select CSV file
  ☑ Preview shows correct data
  ☑ Click Import
  ☑ Data appears in table
  ☑ No duplicates
  ☑ All fields populated

// 3. localStorage PERSISTENCE
  ☑ Add new data
  ☑ Refresh page (F5)
  ☑ Data still there
  ☑ Edit some data
  ☑ Refresh page again
  ☑ Changes preserved

// 4. CLEAR DATA SAFELY
  ☑ Export CSV first (backup)
  ☑ Test clearing mechanism
  ☑ Can reimport from backup
```

#### ✅ Browser & Mode Compatibility

```javascript
// Test in different browsers
  ☑ Chrome/Chromium
  ☑ Firefox
  ☑ Edge
  ☑ Safari (if Mac available)

// Test in different modes
  ☑ Standalone (open HTML file directly)
  ☑ SharePoint (if access available)
  ☑ HTTP server (if deployed)

// Test responsive
  ☑ Full screen (1920x1080)
  ☑ Laptop (1366x768)
  ☑ Tablet (768x1024)
  ☑ Mobile (375x667)
```

#### ✅ Error Handling

```javascript
// Test error scenarios

// 1. Invalid inputs
  ☑ Submit empty form → validation error
  ☑ Submit non-numeric NORAD ID → error
  ☑ Submit invalid date → error
  ☑ Duplicate NORAD ID → handled gracefully

// 2. Edge cases
  ☑ Delete last item → handles
  ☑ Edit deleted item → handles
  ☑ 1000+ satellites → performance ok
  ☑ Very long strings → displays correctly

// 3. Network (SharePoint mode)
  ☑ Connection lost → error message
  ☑ Timeout → retry/cancel option
  ☑ Permission denied → appropriate error
```

#### ✅ Console Check

```javascript
// Before declaring version FIXED, console should be clean

Open F12 → Console tab
  ☑ No red error messages
  ☑ No yellow warnings (ok if minor)
  ☑ Data logging shows expected values
  ☑ No undefined function calls
```

### Phase 4: Document Changes

**After passing all tests, update VERSIONS.md:**

```markdown
### ✅ vX.X.X_FIXED.txt (Feature Name)

**Status:** IN DEVELOPMENT / STABLE
**Release Date:** [Today]
**Based On:** vX.X.X_FIXED

**Changes:**
- [Change 1]
- [Change 2]
- [Change 3]

**Known Issues:**
- [Issue 1: Description, Workaround]
- None known (if working perfectly)

**Tests Passed:**
- ✅ All CRUD operations
- ✅ Display and UI
- ✅ CSV import/export
- ✅ localStorage persistence
- ✅ Browser compatibility
- ✅ No console errors

**When to Use:** [Guidance for users]
```

### Phase 5: GitHub Release

**Create GitHub Release:**

1. Go to [Releases](https://github.com/flaviusviorelbelu-cmd/sensor-survey/releases)
2. Click "Create a new release"
3. Tag: `vX.X.X` (e.g., `v5.6.4`)
4. Title: "Version X.X.X - Description"
5. Description: Copy from VERSIONS.md
6. Upload: `.txt` file as asset
7. Publish

---

## Release Schedule

### When Creating New Versions

**Regular Development:**
- Working on a feature → save as v[X].[X+1].[0]_WIP
- Feature complete → test → v[X].[X+1].[0]_FIXED
- Bug fix to stable → v[X].[X].[Z+1]_FIXED

**Recommended Cadence:**
- Bug fixes: When identified and fixed
- Minor features: Every 2-4 weeks
- Major releases: As needed, with testing

### Retirement Schedule

```
Keep in /scripts/stable/    → Current FIXED version + 1 previous
Move to /scripts/archive/   → All older versions
Move to /scripts/historical/→ Very old versions (6+ months)

Never delete versions - always archive
```

---

## Regression Testing Before Release

### The Two-Version Comparison Test

**Always compare new version against last known good version:**

```
v5.6.3_FIXED (stable)  vs  v5.6.4_FIXED (new)

Side-by-side testing:
1. Open both in separate windows
2. Import same test data to both
3. Run same operations
4. Compare results
5. Look for:
   ✗ Features that worked before but don't now
   ✗ Visual differences
   ✗ Performance degradation
   ✗ Data inconsistencies
```

### Regression Test Data

**Use this standard test dataset:**

```csv
Title,NORAD_ID,COSPAR_ID,Mission_Type,Status,Orbit_Type,Launch_Date,Sensor_Names
Landsat 8,39084,2013-008A,Earth Observation,Operational,SSO,2013-02-11,OLI; TIRS
Sentinel-1,39023,2014-016A,Synthetic Aperture Radar,Operational,Polar,2014-04-03,C-SAR
Copernicus,38833,2015-017A,Remote Sensing,Operational,LEO,2015-06-17,MSI
ISS,25544,1998-067A,Earth Observation,Operational,LEO,1998-11-20,Various
Hubble,20580,1990-037B,Astronomy,Operational,LEO,1990-04-24,ACS; STIS
```

Load same data into both versions and test all operations.

---

## Hotfix Procedure

**If critical issue found in released version:**

```
1. Identify issue in vX.X.X_FIXED
   ↓
2. Create vX.X.(Z+1)_FIXED for the fix
   ↓
3. Make ONLY the fix, nothing else
   ↓
4. Test thoroughly (full checklist)
   ↓
5. Update VERSIONS.md with "Hotfix: [issue]"
   ↓
6. Release vX.X.(Z+1)_FIXED
   ↓
7. Mark previous version as "Use vX.X.(Z+1) instead"
```

**Example:**
```
v5.6.4_FIXED released with visual bug
  ↓ Visual bug critical, delete broken in v5.4.8+
  ↓
v5.6.5_FIXED
  - Fixed visual rendering issue
  - Fixed delete function regression
  - Backported from v5.4.7 delete code
```

---

## Breaking Changes Management

### Detecting Breaking Changes

**If new version breaks existing functionality:**

1. Document which feature broke
2. Note exact version it broke
3. Find last working version
4. Compare code between versions
5. Update VERSIONS.md regression section

### Example: Delete Function Breaking

```
v5.4.7_FIXED - DELETE WORKS ✅
v5.4.8_FIXED - DELETE BROKEN ❌

Breaking change identified: between v5.4.7 and v5.4.8

Code review finds: FormHandler refactoring changed event delegation

Fix: Restore event listener pattern from v5.4.7
Result: v5.6.5_FIXED (hypothetical fix version)
```

---

## Quality Gates

**Do NOT release a version if:**

- ❌ Any test from checklist fails
- ❌ Console has red error messages
- ❌ CRUD operations don't work
- ❌ Data doesn't persist
- ❌ Visual issues present
- ❌ Not tested in multiple browsers
- ❌ Not compared against previous version
- ❌ VERSIONS.md not updated

**Only release when:**

- ✅ ALL checklist items pass
- ✅ Console clean
- ✅ Tested in 2+ browsers
- ✅ Compared against v5.6.3 (baseline)
- ✅ VERSIONS.md updated
- ✅ GitHub release created
- ✅ Someone (or future you) could use it immediately

---

## Quick Reference: From Feature to Release

```
┌─────────────────────────────────────┐
│  Take last stable version           │
│  e.g., v5.6.3_FIXED.txt             │
└────────────┬────────────────────────┘
             ↓
┌─────────────────────────────────────┐
│  Copy & Make Changes                │
│  Save as v5.6.4_WIP.txt (temporary) │
└────────────┬────────────────────────┘
             ↓
┌─────────────────────────────────────┐
│  Run Full Test Checklist            │
│  - CRUD operations                  │
│  - Display & UI                     │
│  - Data persistence                 │
│  - Multiple browsers                │
│  - Console errors?                  │
└────────────┬────────────────────────┘
             ↓
        PASS?  FAIL?
         ↙      ↘
        ✓        Fix issue, go back to tests
        │
        ↓
┌─────────────────────────────────────┐
│  Compare vs Previous Version        │
│  Open both, test same data          │
│  Look for regressions               │
└────────────┬────────────────────────┘
             ↓
        NEW > OLD?  NEW = OLD? NEW < OLD?
          ✓          ⚠️         ❌
          │          Fix        Investigate
          ↓                     Regression
        Continue               Fix, test again
             │
             ↓
┌─────────────────────────────────────┐
│  Update VERSIONS.md                 │
│  - Add feature notes                │
│  - Document known issues            │
│  - List tests passed                │
└────────────┬────────────────────────┘
             ↓
┌─────────────────────────────────────┐
│  Rename v5.6.4_WIP.txt              │
│           ↓                         │
│      v5.6.4_FIXED.txt               │
└────────────┬────────────────────────┘
             ↓
┌─────────────────────────────────────┐
│  Commit to GitHub                   │
│  - Add README.md updates            │
│  - Add VERSIONS.md updates          │
│  - Commit new script file           │
└────────────┬────────────────────────┘
             ↓
┌─────────────────────────────────────┐
│  Create GitHub Release              │
│  - Tag: vX.X.X                      │
│  - Upload .txt file                 │
│  - Add release notes                │
└─────────────────────────────────────┘

                🎉 RELEASED
```

---

## Maintenance Notes

**Keep in mind:**

1. **Version Hygiene**
   - Never delete old versions
   - Archive in github/archive folder
   - Keep historical context

2. **Documentation**
   - Update VERSIONS.md with every release
   - Keep README.md current
   - Document workarounds

3. **Testing**
   - Use same test data always
   - Test in multiple browsers
   - Document unusual behaviors

4. **Communication**
   - Clearly mark stable versions
   - Flag problematic versions
   - Provide upgrade paths

---

**Last Updated:** January 22, 2026  
**Workflow Version:** 1.0  
**For:** Satellite Sensor Survey Project
