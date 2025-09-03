# BigQuery VW_TABLE_CON_DOCTOR_LIST Column Usage Mapping - Speciality & Doctor Seeder

## BigQuery Source Columns Used

### From `VW_TABLE_CON_DOCTOR_LIST` table in BigQuery:

| BigQuery Column       | Maps To                            | Used In                        | Purpose                                        |
| --------------------- | ---------------------------------- | ------------------------------ | ---------------------------------------------- |
| `TIME_KEY`            | Filtering only                     | Data validation                | Time-based data filtering                      |
| `SPECIALITY_DESC`     | `specialities.name`                | Medical Speciality Management  | Healthcare speciality identification           |
| `DOCTOR_NAME`         | `doctors.name`                     | Healthcare Provider Management | Doctor identification and naming               |
| `CEGEDIM_CUSTOMER_ID` | `doctors.refId`                    | Healthcare Provider Management | Unique doctor identifier                       |
| `DRL_CUSTOMER_OCE_ID` | `doctors.customerId`               | Customer Management            | OCE customer identification                    |
| `EMP_DIVISION`        | `division_specialities.divisionId` | Division-Speciality Linking    | Links specialities to pharmaceutical divisions |

## Column Usage Details

### 1. `TIME_KEY` → Data Validation

**Usage**:

- Filters records to latest time period using `Helpers.getLastTimeKey()`
- Ensures data freshness and consistency
- Division-wise processing for large datasets

### 2. `SPECIALITY_DESC` → `specialities.name`

**Processing**:

- **Space Normalization**: Replaces multiple spaces with single spaces
- **Lowercase Conversion**: Converts to lowercase for consistency
- **Trimming**: Removes leading/trailing whitespace
- **Example**: "CONSULTANT PHYSICIANS" becomes "consultant physicians"

### 3. `DOCTOR_NAME` → `doctors.name`

**Processing**:

- Trimmed and converted to lowercase
- Used for doctor identification and display
- Example: "Dr Sumeet Mazumder " becomes "dr sumeet mazumder"

### 4. `CEGEDIM_CUSTOMER_ID` → `doctors.refId`

**Usage**:

- Primary unique identifier for doctors
- Used in createOrUpdate operations
- Links to other systems via customer ID

### 5. `DRL_CUSTOMER_OCE_ID` → `doctors.customerId`

**Usage**:

- OCE (Oracle Customer Experience) system integration
- Customer relationship management identifier
- Links doctors to CRM systems

### 6. `EMP_DIVISION` → Division Context

**Usage**:

- Links specialities to specific pharmaceutical divisions
- Creates division-speciality relationships
- Enables division-specific speciality analytics

## Database Schema

### Specialities Table

```sql
specialities:
  - id (primary key)
  - name (speciality name, normalized)
  - createdAt (record creation)
  - updatedAt (record update)
```

### Doctors Table

```sql
doctors:
  - id (primary key)
  - refId (CEGEDIM customer ID)
  - name (doctor name, normalized)
  - status (active status, default 1)
  - customerId (OCE customer ID)
  - createdAt (record creation)
  - updatedAt (record update)
```

### Division Specialities Table

```sql
division_specialities:
  - id (primary key)
  - divisionId (foreign key to divisions)
  - specialityId (foreign key to specialities)
  - createdAt (record creation)
  - updatedAt (record update)
```

## Where Data is Consumed

### Medical Speciality Analytics

- **Speciality Performance**: Performance tracking by medical speciality
- **Division-Speciality Analysis**: Speciality distribution across divisions
- **Target Speciality Planning**: Strategic planning for specific medical fields
- **Market Segmentation**: Speciality-based market segmentation

### Healthcare Provider Management

- **Doctor Database**: Comprehensive doctor information system
- **Customer Relationship**: Links doctors to CRM systems via OCE IDs
- **Territory Planning**: Doctor distribution for territory assignments
- **Performance Tracking**: Doctor-specific performance analytics

### Business Intelligence

- **Speciality Trends**: Medical speciality market trends
- **Doctor Segmentation**: Doctor categorization by speciality and division
- **Market Analysis**: Healthcare provider market analysis
- **Strategic Planning**: Speciality-focused business strategies

## Data Quality Requirements

### Required Fields

- **TIME_KEY**: Must match expected time period
- **CEGEDIM_CUSTOMER_ID**: Required for doctor identification
- **SPECIALITY_DESC**: Required for speciality creation
- **EMP_DIVISION**: Must exist in divisions table

### Data Validation

- **Division Validation**: Must map to existing pharmaceutical divisions
- **Duplicate Prevention**: Uses firstOrNew to prevent duplicate records
- **Name Normalization**: Consistent naming conventions
- **Status Setting**: All doctors created with active status

### Data Processing Rules

- **Skip Null Specialities**: Skips records with empty speciality descriptions
- **Skip Null Doctors**: Skips records with missing CEGEDIM customer IDs
- **Parallel Processing**: Creates specialities and doctors concurrently
- **Error Tolerance**: Individual record failures don't stop processing

## Data Examples

### Sample BigQuery Record

```typescript
const record = {
  TIME_KEY: BigQueryDate { value: '2024-04-30' },
  SPECIALITY_DESC: 'CONSULTANT PHYSICIANS',
  DOCTOR_NAME: 'Dr Sumeet Mazumder',
  CEGEDIM_CUSTOMER_ID: 217065000083794,
  DRL_CUSTOMER_OCE_ID: 2353066,
  EMP_DIVISION: 'Aqura Sg'
}
```

### Processed Database Records

```sql
-- Specialities
INSERT INTO specialities (name) VALUES ('consultant physicians');

-- Doctors
INSERT INTO doctors (refId, name, status, customerId) VALUES (
  '217065000083794', 'dr sumeet mazumder', 1, 2353066
);

-- Division-Speciality Link
INSERT INTO division_specialities (divisionId, specialityId) VALUES (
  division_id_for_aqura_sg, speciality_id_for_consultant_physicians
);
```

---

**Contact**: Development team for schema changes, Data Ops for BigQuery monitoring, Medical Affairs for speciality classifications
