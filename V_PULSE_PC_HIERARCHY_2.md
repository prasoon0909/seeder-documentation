# BigQuery V_PULSE_PC_HIERARCHY Column Usage Mapping - Region Seeder

## BigQuery Source Columns Used

### From `V_PULSE_PC_HIERARCHY` table in BigQuery:

| BigQuery Column      | Maps To              | Used In             | Purpose                                       |
| -------------------- | -------------------- | ------------------- | --------------------------------------------- |
| `DIVISION_CODE`      | `regions.divisionId` | Division Linking    | Links regions to pharmaceutical divisions     |
| `RSM_ALIGNMENT_NAME` | `regions.name`       | Regional Management | Regional territory name identification        |
| `RSM_ALIGNMENT_ID`   | `regions.code`       | Regional Management | Regional territory code/identifier            |
| `EMP_DESIGNATION`    | Filtering (PSR only) | Employee Filtering  | Ensures region creation from PSR records only |

## Column Usage Details

### 1. `DIVISION_CODE` → `divisionId`

**Usage**:

- Links regions to specific pharmaceutical divisions
- Validates against existing divisions table via refId
- Ensures regions are created within valid division context

### 2. `RSM_ALIGNMENT_NAME` → `name`

**Processing**:

- Converted to lowercase and trimmed for consistency
- Used as the human-readable region name
- Example: "DRL_35_RSM_Pune" becomes region name

### 3. `RSM_ALIGNMENT_ID` → `code`

**Usage**:

- Unique identifier for the region
- Used as primary lookup key for region identification
- Links to employee hierarchy and territory management

### 4. `EMP_DESIGNATION` → Filtering Logic

**Usage**:

- **Filter Condition**: Only processes records where `EMP_DESIGNATION = 'PSR'`
- Ensures region creation from field-level employee data
- Prevents duplicate region creation from management hierarchy

## Data Processing Flow

### Region Creation Process

```typescript
// Validates division exists
const division = divisions[divisionCode]
  ? divisions[divisionCode][0]
  : { id: 0 };
if (division.id === 0) continue;

// Processes region name
const regionName = region?.RSM_ALIGNMENT_NAME?.toLowerCase()?.trim();
if (!regionName) continue;

// Creates or finds existing region
await repo.firstOrNew(
  { code: regionCode, divisionId: division.id },
  { name: regionName },
);
```

### Regional Territory Structure

- **RSM Level**: Regional Sales Manager territories
- **Division Based**: Each region belongs to specific pharmaceutical division
- **Hierarchical**: Regions form part of larger organizational structure
- **Territory Management**: Geographic/alignment-based territory definition

## Database Schema

### Regions Table

```sql
regions:
  - id (primary key)
  - code (RSM alignment ID, unique per division)
  - name (RSM alignment name, lowercase)
  - divisionId (foreign key to divisions)
  - createdAt (record creation)
  - updatedAt (record update)
```

## Where Data is Consumed

### Territory Management

- **Sales Territory Definition**: Geographic region boundaries for sales teams
- **Employee Assignment**: Links employees to specific regional territories
- **Performance Analytics**: Regional performance tracking and comparison
- **Resource Allocation**: Territory-based resource distribution

### Management Hierarchy

- **RSM Territory Mapping**: Links Regional Sales Managers to territories
- **Hierarchical Reporting**: Region-based management reporting structure
- **Performance Tracking**: Regional sales performance metrics
- **Territory Coverage**: Geographic coverage analysis

### Business Intelligence

- **Regional Analytics**: Performance comparison across regions
- **Market Analysis**: Regional market penetration and growth
- **Territory Optimization**: Regional boundary and resource optimization
- **Competitive Analysis**: Regional competitive positioning

## Regional Structure Context

### Organizational Hierarchy

```
Division (e.g., Zenura - Code 35)
├── NSM Territory (DRL_35_NSM_Hyderabad)
    ├── SM Territory (DRL_35_SM_Mumbai)
        ├── RSM Territory (DRL_35_RSM_Pune) ← REGIONS TABLE
            ├── ASM Territory (DRL_35_ASM_Kolhapur)
                └── PSR Employees
```

## Data Quality Requirements

### Required Fields

- **DIVISION_CODE**: Must exist in divisions table via refId
- **RSM_ALIGNMENT_NAME**: Must not be null or empty after trimming
- **RSM_ALIGNMENT_ID**: Unique regional identifier
- **EMP_DESIGNATION**: Must equal 'PSR' for processing

### Validation Logic

- **Division Validation**: Skips regions with invalid division codes
- **Name Validation**: Skips regions with empty/null names
- **PSR Filtering**: Only processes PSR-level employee records
- **Duplicate Prevention**: Uses firstOrNew to avoid duplicate regions

### Data Consistency

- **Unique Constraints**: Code + divisionId combination must be unique
- **Name Standardization**: Lowercase and trimmed region names
- **Division Linking**: All regions must link to valid divisions

## Data Examples

### Sample BigQuery Record

```typescript
const record = {
  DIVISION_CODE: 35,
  RSM_ALIGNMENT_NAME: 'DRL_35_RSM_Pune',
  RSM_ALIGNMENT_ID: '50051569',
  EMP_DESIGNATION: 'PSR',
};
```

### Processed Database Record

```sql
INSERT INTO regions (code, name, divisionId) VALUES (
  '50051569',
  'drl_35_rsm_pune',
  division_id_for_code_35
)
```
