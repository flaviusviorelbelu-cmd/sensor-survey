# Satellite Sensor Survey

A web-based application for managing satellite and sensor data, designed to work both standalone (localStorage) and integrated with SharePoint.

**Current Version:** v5.6.4 (In Development)  
**Current Stable:** v5.6.3 FIXED  
**Repository:** [GitHub](https://github.com/flaviusviorelbelu-cmd/sensor-survey)

## 📋 Version History & Tracking

### Active Versions

| Version | Status | Key Features | Issues | Last Updated |
|---------|--------|--------------|--------|--------------|
| v5.6.4_FIXED.txt | 🔧 In Development | UI/UX improvements, visual refinements | Visual display issues | Current |
| v5.6.3_FIXED.txt | ✅ Stable | Critical UX fixes, pagination working | None known | Jan 2026 |
| v5.6.2_FIXED.txt | 📦 Archive | Pagination implementation | Stable | - |
| v5.6.1_FIXED.txt | 📦 Archive | Tab filtering fixes | Stable | - |

### Regression Tracking

| Feature | Last Working | Broken Since | Status | Fix ETA |
|---------|--------------|--------------|--------|---------|
| Delete Function | v5.4.7 | v5.4.8 | 🔴 Broken | Investigating |
| Visual Display | v5.6.3 | v5.6.4 | 🟡 Degraded | In Progress |

## 🚀 Quick Start

### Standalone Mode (localStorage)
1. Download the latest `script_v5.6.3_FIXED.txt` or `script_v5.6.4_FIXED.txt`
2. Rename to `.html` and open in browser
3. Data persists in browser storage

### SharePoint Integration
1. Deploy to SharePoint as a web part
2. Create lists: `Satellite_Fixed` and `Sensor`
3. Application automatically detects SharePoint context

## 📁 Repository Organization

```
scripts/
├── stable/
│   ├── script_v5.6.3_FIXED.txt          # Current stable version
│   └── script_v5.6.2_FIXED.txt          # Previous stable
├── development/
│   └── script_v5.6.4_FIXED.txt          # Current work-in-progress
├── archive/
│   ├── working/                          # Versions with known working features
│   │   ├── script_v5.4.7_FIXED.txt      # Last version with delete working
│   │   ├── script_v5.5.6_FIXED.txt      # Last version with detail view
│   │   └── ...
│   └── historical/                       # Older versions for reference
│       ├── script_v5.2_with_csv.txt
│       ├── script_v5.3_FIXED.txt
│       └── ...
├── releases/                             # Tagged releases
│   └── v5.6.3_stable/
│       ├── script.html
│       └── CHANGELOG.md
└── docs/
    ├── VERSIONS.md                       # Detailed version notes
    ├── TROUBLESHOOTING.md               # Known issues & fixes
    └── MIGRATION_GUIDE.md               # Upgrading versions
```

## 🐛 Known Issues

### v5.6.4 (Current)
- ❌ **Visual Display Issues** - Some UI elements not rendering correctly
  - **Workaround:** Switch to v5.6.3 until resolved
  - **Affected Elements:** [Check screenshot in issue #X]

### Historical
- ❌ **Delete Function Broken** (v5.4.8+)
  - **Last Working:** v5.4.7
  - **Cause:** Investigating form handler refactoring
  - **Workaround:** Use v5.4.7 for delete operations

## 🔄 Version Comparison Strategy

### How to Test New Features
1. Make changes to development version
2. Test both modes (localStorage & SharePoint)
3. Verify against regression list
4. Tag as `script_v[x.x.x]_FIXED.txt` once validated
5. Update VERSIONS.md with notes

### How to Track Regressions
1. Before updating: Test all CRUD operations
2. Document baseline in VERSIONS.md
3. After update: Run same tests
4. If issue found, note in regression table
5. Reference last working version

## 💾 Data Backup Strategy

Always backup before upgrading:
```bash
# Export data before version change
- Use "Export" button to save current data as CSV
- Store CSV with timestamp
- Name: satellite_data_[date]_v[version].csv
```

## 📝 File Naming Convention

All versions follow this pattern:
```
script_v[MAJOR].[MINOR].[PATCH]_[STATUS].txt

Examples:
- script_v5.6.4_FIXED.txt      (Development)
- script_v5.6.3_FIXED.txt      (Stable)
- script_v5.4.7_FIXED.txt      (Archive - Last delete working)
- script_v5.2_with_csv.txt     (Legacy)
```

## 🛠️ Troubleshooting Guide

### Delete function not working
- **Try:** Switch to v5.4.7 to confirm it's a regression
- **Check:** Browser console for JavaScript errors
- **Investigate:** Form handler class changes between v5.4.7 and v5.4.8

### Visual/Display issues
- **Try:** Hard refresh (Ctrl+Shift+R or Cmd+Shift+R)
- **Check:** Browser compatibility (Chrome, Edge, Firefox)
- **Try:** Switch to v5.6.3 to isolate UI issue

### Data not persisting
- **Check:** Browser localStorage is enabled
- **Check:** SharePoint context (if integrated)
- **Try:** Export/Import via CSV as backup

## 📊 Project Statistics

| Metric | Count |
|--------|-------|
| Total Versions | 20+ |
| Satellites | 59 |
| Sensors | 24 |
| Current Users | TBD |

## 📚 Documentation

- [Detailed Version Notes](./docs/VERSIONS.md)
- [Architecture Overview](./docs/ARCHITECTURE.md)
- [SharePoint Integration Guide](./docs/SHAREPOINT.md)
- [API Documentation](./docs/API.md)

## 🤝 Contributing

When working on versions:
1. Always work from a stable version
2. Test thoroughly before tagging as FIXED
3. Document all changes in VERSIONS.md
4. Maintain changelog format
5. Tag regressions explicitly

## 📞 Support

For issues with specific versions:
- Check TROUBLESHOOTING.md
- Review version notes in VERSIONS.md
- Compare with last known working version
- Reference regression tracking table

---

**Last Updated:** January 22, 2026  
**Maintained by:** @flaviusviorelbelu-cmd
