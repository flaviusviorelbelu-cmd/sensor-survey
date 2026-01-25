# Sensor List HTTP 400 Error - Troubleshooting Guide

## Current Status
✅ **Satellites** loading successfully (30 items from 'Satellite_Fixed' list)
❌ **Sensors** failing with HTTP 400 error

## Root Cause
The SharePoint API call to load from the 'Sensor' list is returning HTTP 400 (Bad Request).

This typically means:
1. **List name is incorrect** - Check exact name in SharePoint (case-sensitive)
2. **Column names don't exist** - One or more of these columns missing:
   - `Title`, `Sensor_Type`, `Description`, `N_of_Bands`
   - `Resolution_Min_m`, `Resolution_Max_m`, `Swath_Min_km`, `Swath_Max_km`, `NORAD_ID`
3. **API query syntax error** - Malformed REST call

## How to Fix

### Step 1: Find the Correct Sensor List Name
1. Go to your SharePoint site: `http://orionsps/s/1/classic`
2. Click **Site Contents** (or Settings → Site Contents)
3. Look for your sensors list
4. **Note the EXACT name** (including spelling, spaces, special characters)

### Step 2: Verify Column Names
1. Click on your Sensor list
2. Go to **List Settings** (or List → List Settings)
3. Check the **Columns** section
4. Compare with required columns (see above)
5. If column names are different, note them

### Step 3: Update the Code
If your list or columns are different, edit the script:

Find this section in the HTML:
```javascript
async function loadSensors() {
    const url = config.siteUrl +
        `/_api/web/lists/getbytitle('Sensor')/items?$select=ID,Title,Sensor_Type,Description,...
```

**Change 'Sensor' to your actual list name:**
```javascript
`/_api/web/lists/getbytitle('YourActualListName')/items?...
```

**And update the $select columns to match your list:**
```javascript
?$select=ID,Title,YourSensorTypeColumn,YourDescriptionColumn,...
```

### Step 4: Common Issues & Solutions

| Error | Likely Cause | Solution |
|-------|-------------|----------|
| HTTP 400 | List name wrong | Check SharePoint Site Contents for exact name |
| HTTP 400 | Column doesn't exist | Check List Settings → Columns for correct names |
| HTTP 404 | List doesn't exist | Verify list is in this site (not subsites) |
| HTTP 403 | No permissions | Verify you have access to the list |

### Step 5: Test
1. Open browser Developer Tools (F12)
2. Open Console tab
3. Reload the page
4. Look for messages like:
   - ✅ `Loaded X sensors from SharePoint` = Success
   - ❌ `Error loading sensors: Error: HTTP 400` = Problem (see above)
   - ⚠️ `Using fallback...` = App continuing without sensors

## Temporary Workaround (v5.8.5+)

The app now has **error recovery** - if Sensor list fails, it will:
- Continue loading satellites (which ARE working)
- Use local demo data for sensors as fallback
- Log detailed error messages for troubleshooting

This allows you to use the app with satellites while fixing the sensor issue.

## Getting Help

When reporting this issue, include:
1. Exact sensor list name from Site Contents
2. Console error message (copy from F12 → Console)
3. Your actual column names (from List Settings)
4. SharePoint version (Online or 2019?)
