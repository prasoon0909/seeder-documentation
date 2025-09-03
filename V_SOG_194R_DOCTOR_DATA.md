# BigQuery V_SOG_194R_DOCTOR_DATA Column Usage Mapping

## BigQuery Source Columns Used

### From `V_SOG_194R_DOCTOR_DATA` table in BigQuery:

| BigQuery Column       | Maps To                | Used In             | Purpose                            |
| --------------------- | ---------------------- | ------------------- | ---------------------------------- |
| `CEGEDIM_CUSTOMER_ID` | `doctor_data.doctorId` | Doctor Matching     | Healthcare provider identification |
| `TOTAL_VALUE`         | `doctor_data.one94R`   | Financial Analytics | 194R total value tracking          |

## Column Usage Details

### 1. `CEGEDIM_CUSTOMER_ID` → Doctor Matching

**Usage**:

- Primary healthcare provider identifier
- Links to doctors table via refId
- Essential for updating correct doctor records

### 2. `TOTAL_VALUE` → `one94R`

**Usage**:

- Financial value associated with 194R classification
- Updates existing doctor_data records with 194R total values
- Used for pharmaceutical financial analytics and compliance tracking

## Data Processing Flow

### 194R Value Update Process

```typescript
// Updates existing doctor_data records with 194R values
await this.doctorLib.doctorDataRepo.updateWhere(
  {
    doctorId: doctor.id,
  },
  {
    one94R: total,
  },
);
```

### Doctor Lookup

```typescript
// Pre-loaded doctor mapping for efficient lookups
const doctor = await this.getDoctor(record?.CEGEDIM_CUSTOMER_ID);
if (doctor.id === 0) continue; // Skip invalid doctors
```

## Database Updates

### Target Table: doctor_data

**Updated Field**: `one94R`
**Update Logic**:

- Matches on doctorId only (updates all doctor_data records for the doctor)
- Updates one94R field with TOTAL_VALUE from BigQuery
- Preserves all other existing data

### Matching Criteria

```sql
WHERE doctorId = matched_doctor_id
```

## What is 194R?

### Pharmaceutical Context

- **194R Classification**: Likely refers to a specific pharmaceutical regulation or classification system
- **Financial Tracking**: Tracks monetary values associated with this classification
- **Compliance Monitoring**: Used for regulatory compliance and reporting
- **Doctor Association**: Links specific financial values to healthcare providers

### Business Applications

- **Regulatory Reporting**: Compliance with pharmaceutical regulations
- **Financial Analytics**: Doctor-specific financial tracking
- **Audit Trail**: Maintains record of 194R-related transactions
- **Performance Metrics**: Financial performance indicators per doctor

## Where Data is Consumed

### Financial Analytics

- **194R Compliance Reports**: Regulatory compliance tracking
- **Doctor Financial Profiles**: Financial association with healthcare providers
- **Audit and Compliance**: Regulatory audit trail maintenance
- **Performance Analysis**: Financial performance metrics

### Regulatory Systems

- **Compliance Dashboards**: 194R regulation compliance monitoring
- **Financial Reporting**: Regulatory financial reporting requirements
- **Audit Preparation**: Data for regulatory audits
- **Risk Management**: Financial risk assessment per doctor

### Business Intelligence

- **Financial KPIs**: 194R-related key performance indicators
- **Doctor Segmentation**: Financial-based doctor categorization
- **Trend Analysis**: 194R value trends over time
- **Comparative Analysis**: Doctor comparison based on 194R values

## Data Quality Requirements

### Required Fields

- **CEGEDIM_CUSTOMER_ID**: Must exist in doctors table
- **TOTAL_VALUE**: Numeric value for 194R classification

### Validation Logic

- **Doctor Validation**: Skips records with non-existent doctors
- **Value Processing**: Accepts numeric TOTAL_VALUE (can be decimal)
- **Bulk Updates**: Updates all doctor_data records for matched doctor

### Data Consistency

- **Existing Records**: Only updates existing doctor_data combinations
- **Preservation**: Maintains all other doctor_data fields unchanged
- **Overwrite Logic**: Overwrites existing one94R values with new data

### Processing Metrics

- **Total Records**: Number of 194R records in BigQuery
- **Valid Updates**: Records with valid doctor matches
- **Skipped Records**: Records with invalid doctor references

## Data Examples

### Sample Record Structure

```typescript
const record = {
  CEGEDIM_CUSTOMER_ID: 25540000010548,
  TOTAL_VALUE: 501.06,
};
```

### Database Update

```sql
UPDATE doctor_data
SET one94R = 501.06
WHERE doctorId = (SELECT id FROM doctors WHERE refId = '25540000010548')
```
