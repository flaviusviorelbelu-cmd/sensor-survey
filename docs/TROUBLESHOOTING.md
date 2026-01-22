# Troubleshooting Guide

## Quick Diagnostics Dashboard

Open browser console (F12 or Right-click → Inspect → Console) and run:

```javascript
// Quick Status Check
console.log(
  'Version Status Check:\n',
  'State loaded:', typeof state !== 'undefined',
  'Satellites count:', state?.satellites?.length || 0,
  'Sensors count:', state?.sensors?.length || 0,
  'In SharePoint:', config?.inSharePoint || false,
  'LocalStorage available:', typeof(localStorage) !== 'undefined'
);

// Check Delete Button
console.log('Delete buttons found:', document.querySelectorAll('[data-action="delete"]').length);

// Check Event Listeners
console.log('Form:', document.querySelector('#add-satellite-form') ? 'Found' : 'Missing');
console.log('Modal:', document.querySelector('#add-satellite-modal') ? 'Found' : 'Missing');
```

---

## Issue #1: Delete Function Not Working

### Symptoms
- Click delete button, nothing happens
- No confirmation dialog appears
- No error in console
- Data not deleted

### Root Cause Analysis

**Status:** Regression since v5.4.8  
**Last Working:** v5.4.7  
**Affected Versions:** v5.5.x, v5.6.x  

### Step 1: Verify You're On Affected Version

```javascript
// Check which version
if (typeof deleteSatellite === 'function') {
  console.log('deleteSatellite function exists');
  // Try to see function code
  console.log(deleteSatellite.toString().substring(0, 200));
}
```

### Step 2: Test Delete Button Detection

```javascript
// Are delete buttons being created?
const satellite = state.satellites[0];
if (satellite) {
  console.log('Sample satellite:', satellite);
  
  // Manually trigger delete
  if (typeof deleteSatellite === 'function') {
    console.log('Attempting delete of ID:', satellite.ID);
    // Don't actually run this, just check if function exists
  }
}

// Check if delete buttons exist in DOM
const deleteButtons = document.querySelectorAll('[data-action="delete"]');
console.log('Delete buttons in DOM:', deleteButtons.length);
if (deleteButtons.length > 0) {
  console.log('First delete button:', deleteButtons[0].getAttribute('data-id'));
}
```

### Step 3: Check Event Listeners

```javascript
// Method 1: Check table for event delegation
const table = document.querySelector('#satellite-table');
if (table) {
  console.log('Table found, checking listeners...');
  // Note: Can't directly see listeners, but we can test clicking
}

// Method 2: Manually test click handler
const deleteBtn = document.querySelector('[data-action="delete"]');
if (deleteBtn) {
  console.log('Testing delete button click event...');
  const evt = new MouseEvent('click', { bubbles: true });
  // Don't fire, just check if it would work
  console.log('Event would fire on:', deleteBtn);
}
```

### Step 4: Compare v5.4.7 vs v5.6.4

**What to Check in Code:**

```javascript
// In v5.4.7 (WORKING):
// Look for something like:
document.addEventListener('click', function(e) {
  if (e.target.dataset.action === 'delete') {
    deleteSatellite(parseInt(e.target.dataset.id));
  }
});

// In v5.6.4 (BROKEN):
// This might be missing or attached differently
// Or the SatelliteFormHandler class might be interfering
```

### Solution 1: Quick Workaround (Use Two Versions)

**Delete data using v5.4.7:**

1. **Backup data from v5.6.3:**
   ```
   - Open v5.6.3_FIXED.txt
   - Click Export
   - Save as: satellite_data_v5.6.3_backup.csv
   ```

2. **Switch to v5.4.7:**
   ```
   - Close v5.6.3
   - Open v5.4.7_FIXED.txt as .html
   - Click Import
   - Select satellite_data_v5.6.3_backup.csv
   - Wait for data to load
   ```

3. **Delete satellites:**
   ```
   - Delete button should work here
   - Click delete, confirm
   - Repeat for all satellites to delete
   ```

4. **Export cleaned data:**
   ```
   - Click Export
   - Save as: satellite_data_cleaned.csv
   ```

5. **Back to v5.6.3:**
   ```
   - Open v5.6.3_FIXED.txt
   - Import satellite_data_cleaned.csv
   - Verify deletions were successful
   ```

### Solution 2: Fix the Code (For v5.6.4+)

**Debug the deleteSatellite function:**

```javascript
// Add this to console to test
function testDelete(id) {
  console.log('Delete requested for ID:', id);
  
  if (!confirm('Are you sure?')) {
    console.log('Delete cancelled by user');
    return;
  }
  
  console.log('Before delete, count:', state.satellites.length);
  state.satellites = state.satellites.filter(s => s.ID !== id);
  console.log('After delete, count:', state.satellites.length);
  
  // Save and refresh
  saveToLocalStorage();
  filterAndDisplayData();
  closeAddSatelliteModal();
  
  console.log('Delete completed');
}

// Test it
testDelete(1); // Replace 1 with actual satellite ID
```

**If test works, add to code:**

The issue is likely that the delete event listener is not attached. In v5.6.4, check:

1. Is `deleteSatellite` function defined?
2. Are delete buttons being rendered in HTML?
3. Is event delegation working?

```javascript
// Add this at the end of your script to ensure event listener works
document.addEventListener('click', function(event) {
  // Check if delete button was clicked
  if (event.target.dataset?.action === 'delete') {
    const satelliteId = parseInt(event.target.dataset.id);
    console.log('Delete clicked for:', satelliteId);
    deleteSatellite(satelliteId);
  }
});
```

### Prevention: When Fixing

- Always test delete before marking version as FIXED
- Compare event listener logic with v5.4.7
- Use console to verify:
  - Delete buttons exist in DOM
  - Click events fire
  - Data actually deletes
  - Display updates

---

## Issue #2: Visual Display Problems (v5.6.4)

### Symptoms
- Tables not fully visible
- Columns appear truncated or cut off
- Detail panel not showing
- Statistics cards not updating
- Reference screenshot shows partial table display

### Analysis from Screenshot

Observations:
- Only ~3 columns visible of satellite table
- Table appears to be width-constrained
- Possible overflow: hidden issue
- May be modal/container issue

### Step 1: Inspect CSS Issues

```javascript
// Check table element
const table = document.querySelector('#satellite-table');
if (table) {
  const styles = window.getComputedStyle(table);
  console.log('Table Display:', styles.display);
  console.log('Table Width:', styles.width);
  console.log('Table Overflow:', styles.overflow);
  console.log('Table Max-height:', styles.maxHeight);
}

// Check parent container
const container = table?.parentElement;
if (container) {
  const styles = window.getComputedStyle(container);
  console.log('Container Width:', styles.width);
  console.log('Container Overflow:', styles.overflow);
  console.log('Container Display:', styles.display);
}
```

### Step 2: Check Element Visibility

```javascript
// Are elements hidden?
console.log('Table hidden:', table?.style.display === 'none');
console.log('Table visible:', table?.offsetHeight > 0);

// Check modal
const modal = document.querySelector('#add-satellite-modal');
console.log('Modal visible:', modal?.style.display !== 'none');

// Check detail panel
const detail = document.querySelector('.detail-panel');
console.log('Detail visible:', detail?.style.display !== 'none');
```

### Step 3: Check CSS Variables

```javascript
// Get all CSS variables
const root = document.documentElement;
const allVars = getComputedStyle(root);

// Sample critical ones
console.log('Primary color:', allVars.getPropertyValue('--color-primary'));
console.log('Max width:', allVars.getPropertyValue('--max-width'));
console.log('Table width:', allVars.getPropertyValue('--table-width'));
```

### Solution 1: Force Display (Temporary)

```javascript
// In browser console, run this:

// Force tables to display
const style = document.createElement('style');
style.innerHTML = `
  #satellite-table { 
    display: table !important; 
    width: 100% !important; 
    overflow: visible !important; 
  }
  .detail-panel {
    display: block !important;
    overflow: visible !important;
    max-height: none !important;
  }
  .statistics-container {
    display: grid !important;
    grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)) !important;
  }
`;
document.head.appendChild(style);
console.log('CSS override applied. Check if tables now visible.');
```

**If this fixes it:**
- Problem is CSS overflow/display
- Solution: Update CSS in v5.6.4
- Look for `overflow: hidden` or `max-height: 0` on these elements

### Solution 2: Check HTML Structure

```javascript
// Verify HTML elements exist
console.log('Satellite table:', document.querySelector('#satellite-table')?.tagName);
console.log('Sensor table:', document.querySelector('#sensor-table')?.tagName);
console.log('Detail panel:', document.querySelector('.detail-panel')?.tagName);

// Check table content
const rows = document.querySelectorAll('#satellite-table tbody tr');
console.log('Table rows:', rows.length);
if (rows.length > 0) {
  console.log('First row HTML:', rows[0].innerHTML.substring(0, 200));
}
```

### Solution 3: Debug CSS Issues

```javascript
// Find what's hiding things
function findIssues() {
  const table = document.querySelector('#satellite-table');
  let element = table;
  
  while (element && element !== document.body) {
    const styles = window.getComputedStyle(element);
    if (styles.display === 'none' || styles.visibility === 'hidden') {
      console.log('HIDDEN:', element.className, 'display:', styles.display);
    }
    if (styles.overflow === 'hidden' && styles.height === '0px') {
      console.log('COLLAPSED:', element.className, 'height:', styles.height);
    }
    element = element.parentElement;
  }
}

findIssues();
```

### Prevention: CSS Best Practices

When updating CSS in v5.6.4:

- ✅ Check for `overflow: hidden` + `max-height: 0`
- ✅ Verify `display: block` on tables
- ✅ Test responsive breakpoints
- ✅ Don't use `visibility: hidden` on containers
- ✅ Check modal z-index doesn't overlap
- ✅ Verify parent container width

---

## Issue #3: Data Not Persisting

### Symptoms
- Data disappears after refresh
- Import works, but doesn't save
- localStorage shows no data

### Step 1: Check Storage

```javascript
// Check localStorage
const stored = localStorage.getItem('satelliteData');
console.log('LocalStorage has data:', !!stored);
if (stored) {
  const data = JSON.parse(stored);
  console.log('Stored satellites:', data.length);
}

// Check state
console.log('State satellites:', state.satellites.length);
```

### Step 2: Test Save

```javascript
// Manually save
if (typeof saveToLocalStorage === 'function') {
  saveToLocalStorage();
  console.log('Save function called');
  
  // Verify it saved
  const stored = localStorage.getItem('satelliteData');
  console.log('After save, localStorage has:', stored ? 'data' : 'nothing');
}
```

### Step 3: Check SharePoint Mode

```javascript
// If in SharePoint
if (config.inSharePoint) {
  console.log('SharePoint mode detected');
  console.log('Site URL:', config.siteUrl);
  console.log('List name:', config.satelliteList);
  
  // This might be why localStorage not working
}
```

### Solution

```javascript
// Force save and reload
saveToLocalStorage();
window.location.reload();
// Data should persist after reload
```

---

## Issue #4: CSV Import Not Working

### Symptoms
- Import button doesn't respond
- File selected but nothing happens
- Modal doesn't close after import
- Data not imported

### Debug Steps

```javascript
// Check if import function exists
console.log('handleCSVFileSelect:', typeof handleCSVFileSelect);
console.log('handleCSVImport:', typeof handleCSVImport);

// Check file input
const fileInput = document.querySelector('input[type="file"]');
console.log('File input found:', !!fileInput);
if (fileInput) {
  console.log('File input accepts:', fileInput.accept);
}

// Test import manually
if (typeof handleCSVFileSelect === 'function') {
  console.log('handleCSVFileSelect exists, can test');
}
```

### Solution

Always export data before importing to prevent loss:

```javascript
// Before any import
if (typeof exportData === 'function') {
  exportData(); // Creates backup CSV
  console.log('Backup created');
}
```

---

## Debugging Commands Reference

```javascript
// Display current state
console.table(state.satellites);
console.table(state.sensors);

// Test a function
deleteSatellite(1);      // Test delete
saveSatellite({...});    // Test save
filterAndDisplayData();  // Refresh display

// Clear state (dangerous!)
localStorage.clear();    // Wipes localStorage
state.satellites = [];   // Clear state array

// Check configurations
console.log('Config:', config);
console.log('State:', state);

// Find errors
console.log(localStorage);
Object.keys(window).filter(k => k.includes('error')); // Find error objects
```

---

## When to Downgrade vs Fix

| Situation | Action | Why |
|-----------|--------|-----|
| Delete not working, need immediate fix | Use v5.4.7 workaround | Delete works in v5.4.7 |
| Visual issues on v5.6.4 | Downgrade to v5.6.3 | v5.6.3 is stable |
| Need latest features | Fix v5.6.4 | Worth effort for current version |
| Just want to work | Use v5.6.3 | Stable and mostly functional |
| Debugging features | Use v5.4.7 + v5.6.3 comparison | Better understanding |

---

## Browser Console Reference

**Open Console:**
- Windows/Linux: F12 or Ctrl+Shift+I
- Mac: Cmd+Option+I
- Right-click → Inspect → Console tab

**Common Commands:**
```javascript
console.log()      // Output text
console.table()    // Output data as table
console.error()    // Output error
console.time()     // Start timer
console.timeEnd()  // End timer
debuger            // Pause execution
```

---

## Report Issues

When reporting a problem, include:

1. **Version being used:** `script_vX.X.X_FIXED.txt`
2. **Browser:** Chrome, Firefox, Edge, Safari
3. **Console output:** Copy relevant console.log results
4. **Steps to reproduce:**
   - What you did
   - What happened
   - What you expected
5. **Data state:** Number of satellites/sensors
6. **Mode:** localStorage or SharePoint

---

**Last Updated:** January 22, 2026
