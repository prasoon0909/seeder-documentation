# BigQuery VW_TABLE_CON_DOCTOR_LIST Column Usage Mapping

## BigQuery Source Columns Used

### From `VW_TABLE_CON_DOCTOR_LIST` table in BigQuery:

| BigQuery Column         | Maps To                        | Used In                            | Purpose                              |
| ----------------------- | ------------------------------ | ---------------------------------- | ------------------------------------ |
| `TIME_KEY`              | Filtering only                 | Data validation                    | Time-based data filtering            |
| `EMP_POSITION_CD`       | `doctor_data.tmPositionId`     | Employee Management, Analytics     | Employee position identification     |
| `CEGEDIM_CUSTOMER_ID`   | `doctor_data.doctorId`         | Healthcare Provider Management     | Doctor identification and linking    |
| `EMP_DIVISION`          | `doctor_data.divisionId`       | Division Management, SOG Analytics | Pharmaceutical division assignment   |
| `SPECIALITY_DESC`       | `doctor_data.specialityId`     | Medical Speciality Analytics       | Healthcare speciality categorization |
| `CLASSIFICATION_TYPE`   | `doctor_data.classification`   | Doctor Classification Analytics    | Doctor type classification           |
| `SEGMENT_NAME`          | `doctor_data.oceId`            | OCE Management                     | Customer segmentation                |
| `SUPER_CORE_FLAG`       | `doctor_data.divSuperCoreFlag` | Priority Analytics                 | Strategic doctor identification      |
| `DIGICX_FLAG`           | `doctor_data.digiCXFlag`       | Digital Engagement Analytics       | Digital platform engagement          |
| `MEDVOL_FLAG`           | `doctor_data.medvolFlag`       | Medical Volume Analytics           | Medical volume classification        |
| `QUADRANTS_CY`          | `doctor_data.quadrantMedian`   | Performance Analytics              | Doctor performance quadrant          |
| `PRODUCT_SEQ`           | `doctor_data.productSeq`       | Brand Priority Analytics           | Product sequence/priority            |
| `Physical_JCC_count`    | `doctor_data.jcc`              | Contact Analytics                  | Physical contact count               |
| `DOC_AGE`               | `doctor_data.ageGroup`         | Demographic Analytics              | Doctor age group classification      |
| `CIS_SCORE`             | `doctor_data.cis`              | Performance Analytics              | Customer intelligence score          |
| `DOC_GSPED_FLAG`        | `doctor_data.gspFlag`          | Engagement Analytics               | GSPED program participation          |
| `BRAND_NAME`            | `doctor_data.brandId`          | Brand Analytics                    | Brand association                    |
| `DOC_CONSULTATION_FEES` | `doctor_data.consultationFee`  | Financial Analytics                | Consultation fee indicator           |

## Column Usage Details

### 1. `TIME_KEY` → Data Validation

**Usage**:

- Filters records to latest time period using `Helpers.getLastTimeKey()`
- Ensures data freshness and consistency
- Format: 'YYYY-MM-DD'

### 2. `EMP_POSITION_CD` → `tmPositionId`

**Usage**:

- Links to employee management system
- Required for territory and hierarchy mapping
- Validates against existing employees table

### 3. `CEGEDIM_CUSTOMER_ID` → `doctorId`

**Usage**:

- Primary healthcare provider identifier
- Links to doctors table via refId
- Core for all doctor-related analytics

### 4. `EMP_DIVISION` → `divisionId`

**Processing**:

- Converted to lowercase for matching
- Links to pharmaceutical divisions
- Used in SOG dashboard division filtering

### 5. `SPECIALITY_DESC` → `specialityId`

**Processing**:

- Normalized spacing and converted to lowercase
- Maps to medical specialities
- Used in speciality-based analytics

### 6. Flag Fields Processing

**Boolean Conversions**:

- `SUPER_CORE_FLAG`: '1' → true (Strategic doctor)
- `DIGICX_FLAG`: '1' → true (Digital engagement)
- `MEDVOL_FLAG`: '1' → true (Medical volume)
- `DOC_GSPED_FLAG`: '1' → true (GSPED participation)

### 7. Mapping Fields

**Configuration-based Mapping**:

- `QUADRANTS_CY` → `settings.brandQuadrantsMap`
- `PRODUCT_SEQ` → `settings.brandPrioritiesMap`
- `CLASSIFICATION_TYPE` → `settings.classificationMap`
- `DOC_AGE` → `settings.agesMap`

## Data Processing Flow

### Division-based Processing

```typescript
// Processes by division to manage large datasets
SELECT EMP_DIVISION, count(*) FROM table
GROUP BY EMP_DIVISION

// Then processes each division separately
WHERE EMP_DIVISION = 'division_name'
```

### Doctor Data Creation

```typescript
const row: DoctorData$Model = {
  tmId: employee.id,
  divisionId: division.id,
  doctorId: doctor.id,
  specialityId: speciality.id,
  brandId: brand.id,
  // ... 15+ additional fields
  rcpa: calculatedRcpaValue, // From separate RCPA calculation
};
```

### RCPA Calculation

```typescript
// Calculates doctor-brand RCPA from historical data
// Averages Q1 and Q4 data for current performance metric
const tmRcpa = qtr1Average + qtr2Average;
```

## Database Schema

### Doctor Data Table

```sql
doctor_data:
  - id (primary key)
  - ulid (unique identifier)
  - tmId (employee ID)
  - divisionId (pharmaceutical division)
  - doctorId (healthcare provider)
  - brandId (brand association)
  - specialityId (medical speciality)
  - regionId (geographical region)
  - rcpa (calculated performance metric)
  - productSeq (brand priority sequence)
  - quadrantMedian (performance quadrant)
  - divSuperCoreFlag (strategic importance)
  - digiCXFlag (digital engagement)
  - medvolFlag (medical volume)
  - classification (doctor type)
  - ageGroup (age classification)
  - consultationFee (fee indicator)
  - jcc (contact count)
  - cis (intelligence score)
  - gspFlag (program participation)
  - oceId (segment assignment)
  - active (status flag)
```

## Where Data is Consumed

### Analytics Systems

- **SOG Dashboard**: Doctor performance analytics by division/brand
- **Territory Management**: Employee-doctor territory assignments
- **Brand Analytics**: Brand-specific doctor engagement
- **Speciality Analytics**: Medical speciality performance tracking

### Business Intelligence

- **RCPA Analysis**: Doctor prescription performance
- **Digital Engagement**: DigiCX platform usage tracking
- **Contact Analytics**: Physical and digital contact frequency
- **Segmentation**: OCE-based customer segmentation

### Reporting Systems

- **Performance Reports**: Quadrant-based doctor classification
- **Priority Analytics**: Product sequence and brand priorities
- **Demographic Analysis**: Age and classification breakdowns
- **Financial Analytics**: Consultation fee patterns

## Data Quality Requirements

### Required Fields

- **TIME_KEY**: Must match expected time period
- **EMP_POSITION_CD**: Must exist in employees table
- **CEGEDIM_CUSTOMER_ID**: Must exist in doctors table
- **EMP_DIVISION**: Must exist in divisions table

### Optional but Important

- **SPECIALITY_DESC**: Normalized for consistent matching
- **BRAND_NAME**: Processed through WordErrosMap
- **Flag Fields**: Converted to boolean (1 = true, else false)
- **Numeric Fields**: Default to 0 or -1 for invalid values

### Data Validation

- **Duplicate Prevention**: Redis cache prevents duplicate processing
- **Division Filtering**: Only processes known divisions
- **Employee Validation**: Skips records with invalid employees
- **Time Consistency**: Validates TIME_KEY matches expected period
