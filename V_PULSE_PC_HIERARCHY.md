# BigQuery V_PULSE_PC_HIERARCHY Column Usage Mapping

## BigQuery Source Columns Used

### From `V_PULSE_PC_HIERARCHY` table in BigQuery:

| BigQuery Column      | Maps To                  | Used In              | Purpose                          |
| -------------------- | ------------------------ | -------------------- | -------------------------------- |
| `EMP_DIVISION`       | `employees.divisionId`   | Division Management  | Employee division assignment     |
| `EMP_DESIGNATION`    | Filtering (PSR only)     | Employee Filtering   | Role-based processing filter     |
| `NSM_ALIGNMENT_ID`   | `managers.positionCode`  | Management Hierarchy | NSM position identification      |
| `NSM_EMP_CODE`       | `managers.refId`         | Management Hierarchy | NSM employee code                |
| `NSM_EMP_NAME`       | `managers.name`          | Management Hierarchy | NSM manager name                 |
| `NSM_EMAIL_ID`       | `managers.email`         | Management Hierarchy | NSM contact information          |
| `NSM_ALIGNMENT_NAME` | `managers.alignmentName` | Management Hierarchy | NSM territory alignment          |
| `RSM_ALIGNMENT_ID`   | `managers.positionCode`  | Management Hierarchy | RSM position identification      |
| `RSM_EMP_CODE`       | `managers.refId`         | Management Hierarchy | RSM employee code                |
| `RSM_EMP_NAME`       | `managers.name`          | Management Hierarchy | RSM manager name                 |
| `RSM_EMAIL_ID`       | `managers.email`         | Management Hierarchy | RSM contact information          |
| `RSM_ALIGNMENT_NAME` | `managers.alignmentName` | Management Hierarchy | RSM territory alignment          |
| `SM_ALIGNMENT_ID`    | `managers.positionCode`  | Management Hierarchy | SM position identification       |
| `SM_EMP_CODE`        | `managers.refId`         | Management Hierarchy | SM employee code                 |
| `SM_EMP_NAME`        | `managers.name`          | Management Hierarchy | SM manager name                  |
| `SM_EMAIL_ID`        | `managers.email`         | Management Hierarchy | SM contact information           |
| `SM_ALIGNMENT_NAME`  | `managers.alignmentName` | Management Hierarchy | SM territory alignment           |
| `ASM_ALIGNMENT_ID`   | `managers.positionCode`  | Management Hierarchy | ASM position identification      |
| `ASM_EMP_CODE`       | `managers.refId`         | Management Hierarchy | ASM employee code                |
| `ASM_EMP_NAME`       | `managers.name`          | Management Hierarchy | ASM manager name                 |
| `ASM_EMAIL_ID`       | `managers.email`         | Management Hierarchy | ASM contact information          |
| `ASM_ALIGNMENT_NAME` | `managers.alignmentName` | Management Hierarchy | ASM territory alignment          |
| `EMP_POSITION_CODE`  | `employees.tmPositionId` | Employee Management  | Employee position identification |
| `EMPLOYEE_CD`        | `employees.refId`        | Employee Management  | Employee reference code          |
| `EMPLOYEE_NAME`      | `employees.name`         | Employee Management  | Employee full name               |

## Management Hierarchy Structure

### Hierarchy Levels (Top to Bottom)

1. **NSM** (National Sales Manager) - `settings.rolesMap.NSM`
2. **RSM** (Regional Sales Manager) - `settings.rolesMap.RSM`
3. **SM** (Sales Manager) - `settings.rolesMap.SM`
4. **ASM** (Area Sales Manager) - `settings.rolesMap.ASM`
5. **PSR** (Pharmaceutical Sales Representative) - Field employees

## Column Usage Details

### 1. `EMP_DIVISION` → `divisionId`

**Processing**:

- Converted to lowercase and trimmed
- Mapped to existing divisions
- Links all hierarchy levels to same pharmaceutical division

### 2. `EMP_DESIGNATION` → Filtering Logic

**Usage**:

- **Filter Condition**: Only processes records where `EMP_DESIGNATION = 'PSR'`
- Ensures seeder focuses on field-level employees
- Skips other designations to avoid data duplication

### 3. Manager Hierarchy Fields

**Pattern for Each Level (NSM, RSM, SM, ASM)**:

```typescript
// Example for NSM level
ALIGNMENT_ID → managers.positionCode (unique identifier)
EMP_CODE → managers.refId (employee reference)
EMP_NAME → managers.name (manager name)
EMAIL_ID → managers.email (contact email)
ALIGNMENT_NAME → managers.alignmentName (territory/alignment)
```

### 4. Employee Fields

- `EMP_POSITION_CODE` → Primary key for employee records
- `EMPLOYEE_CD` → Employee reference code
- `EMPLOYEE_NAME` → Employee display name

## Data Processing Flow

### Manager Creation (Parallel Processing)

```typescript
// Creates 4 manager levels concurrently
const [nsm, rsm, sm, asm] = await Promise.all([
  createManager(NSM_data, 'NSM'),
  createManager(RSM_data, 'RSM'),
  createManager(SM_data, 'SM'),
  createManager(ASM_data, 'ASM'),
]);
```

### Employee Creation with Hierarchy Links

```typescript
await repo.createOrUpdate(
  {
    tmPositionId: EMP_POSITION_CODE,
  },
  {
    refId: EMPLOYEE_CD,
    divisionId: division.id,
    name: EMPLOYEE_NAME,
    tmPositionName: RSM_ALIGNMENT_ID, // Territory reference
    nsmId: nsm.id, // Links to NSM
    rsmId: rsm.id, // Links to RSM
    smId: sm.id, // Links to SM
    asmId: asm.id, // Links to ASM
    active: 1,
  },
);
```

## Database Schema

### Managers Table

```sql
managers:
  - id (primary key)
  - positionCode (alignment ID from BigQuery)
  - refId (employee code)
  - name (manager name)
  - email (contact email, lowercase)
  - alignmentName (territory alignment)
  - password (default: hashed 'Drl@1234')
  - role (NSM=1, RSM=2, SM=3, ASM=4)
  - divisionId (pharmaceutical division)
  - active (status flag)
```

### Employees Table

```sql
employees:
  - id (primary key)
  - tmPositionId (position code from BigQuery)
  - refId (employee code)
  - divisionId (pharmaceutical division)
  - name (employee name)
  - tmPositionName (territory reference)
  - nsmId (foreign key to managers)
  - rsmId (foreign key to managers)
  - smId (foreign key to managers)
  - asmId (foreign key to managers)
  - active (status flag)
```

## Where Data is Consumed

### Management Hierarchy Analytics

- **SOG Dashboard**: Manager-based filtering and reporting
- **Territory Management**: Hierarchical territory assignments
- **Performance Analytics**: Manager performance tracking
- **Initiative Management**: Management approval workflows

### Employee Management

- **Doctor Assignment**: Employee-doctor territory mapping
- **Activity Tracking**: Employee activity performance
- **RCPA Analytics**: Employee prescription tracking
- **Territory Coverage**: Geographic coverage analysis

### Business Intelligence

- **Hierarchy Reports**: Management structure visualization
- **Performance Metrics**: Manager vs employee performance
- **Territory Analysis**: Coverage and overlap analysis
- **ROI Calculation**: Management efficiency metrics

## Configuration Requirements

### Roles Mapping (`settings.rolesMap`)

```typescript
rolesMap: {
  "NSM": 1,  // National Sales Manager
  "RSM": 2,  // Regional Sales Manager
  "SM": 3,   // Sales Manager
  "ASM": 4   // Area Sales Manager
}
```

## Data Quality Requirements

### Required Fields

- **EMP_DIVISION**: Must exist in divisions table
- **EMP_DESIGNATION**: Must equal 'PSR' for processing
- **EMP_POSITION_CODE**: Unique employee identifier
- **EMPLOYEE_CD**: Employee reference code

### Manager Validation

- **ALIGNMENT_ID**: Required for manager creation
- **EMP_CODE**: Manager employee reference
- **EMP_NAME**: Manager name (trimmed)
- **EMAIL_ID**: Manager email (lowercase, trimmed)

### Data Processing Rules

- **Skip Empty Managers**: Skips manager creation if ALIGNMENT_ID is empty
- **Territory Linking**: Uses RSM_ALIGNMENT_ID as tmPositionName for employees
V_PULSE_PC_HIERARCHY
