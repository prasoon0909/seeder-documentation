# BigQuery V_ALL_EVENTS Column Usage Mapping

## BigQuery Source Columns Used

### From `V_ALL_EVENTS` table in BigQuery:

| BigQuery Column | Maps To | Used In | Purpose |
|-----------------|---------|---------|---------|
| `divisionCode` | `activities.divisionId` | SOG Dashboard, HCP Reports | Pharmaceutical division filtering |
| `cegedimCustomerId` | `activities.doctorId` | SOG Dashboard, HCP Reports | Healthcare provider identification |
| `activityName` | `activities.activityName` | SOG Dashboard, HCP Reports | Marketing initiative matching |
| `sourceActivity` | `activities.sourceActivity` | SOG Dashboard | Activity type categorization |
| `fy` | `activities.fy` | SOG Dashboard, HCP Reports | Financial year filtering |
| `qtr` | `activities.qtr` | HCP Reports | Quarterly analysis |

## Column Usage Details

### 1. `divisionCode` → `divisionId`
**Target Divisions**: 31, 32, 34, 35, 56
**Usage**: 
- SOG Dashboard: `WHERE divisionId = mktInitiative.divisionId`
- HCP Reports: Division-based filtering

### 2. `cegedimCustomerId` → `doctorId`
**Usage**:
- SOG Dashboard: `WHERE doctorId IN (doctor_list)`
- HCP Reports: `WHERE doctorId IN (target_doctors)`
- Key for linking activities to healthcare providers

### 3. `activityName`
**Usage**:
- SOG Dashboard: `WHERE activityName = mktInitiative.name`
- HCP Reports: `WHERE activityName IN (initiative_names)`
- Must match initiative names exactly

### 4. `sourceActivity`
**Mapping**:
- `1` = DIGICX (Digital engagement)
- `2` = SAMPLE AND PROMO (Promotional activities)
- `3` = CAMPAIGNS (Marketing campaigns)
**Usage**: Activity type filtering in SOG Dashboard

### 5. `fy` (Financial Year)
**Usage**:
- HCP Reports: `WHERE fy = financial_year`
- Time-based filtering for reporting

### 6. `qtr` (Quarter)
**Usage**:
- HCP Reports: `SELECT doctorId, activityName, qtr`
- Quarterly performance analysis

## Where Data is Consumed

### SOG Dashboard (`sogDashboard.ts`)
```sql
SELECT * FROM activities 
WHERE doctorId IN (doctor_list)
  AND activityName = 'initiative_name'
  AND divisionId = division_id
  AND sourceActivity = activity_type
```

### HCP Reports (`exportDashboardHcpToCsv.ts`)
```sql
SELECT doctorId, activityName, qtr FROM activities
WHERE fy = financial_year
  AND doctorId IN (target_doctors)
  AND activityName IN (initiative_names)
```

## Data Quality Requirements

- **divisionCode**: Must be in [31, 32, 34, 35, 56]
- **cegedimCustomerId**: Must exist in doctors table
- **activityName**: Must match initiative names exactly
- **sourceActivity**: Must map to valid activity types
- **fy/qtr**: Required for time-based queries
