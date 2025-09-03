# BigQuery VW_TABLE_CON_DOCTOR_LIST (RCPA) Column Usage Mapping

## BigQuery Source Columns Used

### From `VW_TABLE_CON_DOCTOR_LIST` table in BigQuery:

| BigQuery Column       | Maps To                              | Used In                     | Purpose                                           |
| --------------------- | ------------------------------------ | --------------------------- | ------------------------------------------------- |
| `TIME_KEY`            | `year`, `qtr` calculation            | RCPA Analytics              | Time-based RCPA data tracking                     |
| `CEGEDIM_CUSTOMER_ID` | `doctorId` (via doctor lookup)       | Doctor-RCPA Linking         | Links RCPA data to specific doctors               |
| `EMP_DIVISION`        | Division lookup for brand validation | Brand Management            | Ensures brand belongs to correct division         |
| `DRL_RCPA_VALUE`      | `rcpa`                               | RCPA Calculations           | Doctor's RCPA monetary value                      |
| `DRL_RCPA_QTY`        | `rcpaQty`                            | RCPA Analytics              | Doctor's RCPA quantity metrics                    |
| `EMP_POSITION_CD`     | `tmPositionCode`                     | Employee-Doctor Linking     | Links RCPA data to territory managers             |
| `DOC_GSPED_FLAG`      | `gspFlag`                            | Doctor Classification       | GSP (Good Sales Practice) compliance flag         |
| `PRODUCT_SEQ`         | `productSeq` (via priorities map)    | Brand Priority Analytics    | Product sequence priority mapping                 |
| `QUADRANTS_CY`        | `quadrantMedian` (via quadrants map) | Brand Performance Analytics | Brand performance quadrant classification         |
| `BRAND_NAME`          | `brandId` (via brand lookup)         | Brand-RCPA Analytics        | Links RCPA data to specific pharmaceutical brands |

## Column Usage Details

### 1. `TIME_KEY` → Year and Quarter Calculation

**Processing**:

- Format: 'YYYY-MM-DD' (e.g., '2024-04-30')
- **Year Extraction**: `timeKey.split('-')[0]` → `year` field
- **Quarter Calculation**: Month extracted and converted to quarter using `Helpers.getQtrFromMonth(month)`
- **Purpose**: Time-based RCPA trend analysis and quarterly reporting

### 2. `CEGEDIM_CUSTOMER_ID` → Doctor Linking

**Processing**:

- Used as `refId` to lookup doctors in local database
- **Validation**: Records skipped if doctor not found (`doctor.id === 0`)
- **Purpose**: Links RCPA data to specific healthcare providers
- **Critical**: Primary key for doctor-RCPA relationship

### 3. `DRL_RCPA_VALUE` → RCPA Monetary Value

**Processing**:

- Direct mapping to `rcpa` field
- **Null Handling**: Accepts null values for incomplete RCPA data
- **Purpose**: Tracks monetary value of RCPA for doctors
- **Analytics**: Used in revenue calculations and doctor performance metrics

### 4. `DRL_RCPA_QTY` → RCPA Quantity Metrics

**Processing**:

- Direct mapping to `rcpaQty` field
- **Null Handling**: Accepts null values for incomplete quantity data
- **Purpose**: Tracks quantity-based RCPA metrics
- **Analytics**: Used in volume analysis and prescription tracking

### 5. `EMP_POSITION_CD` → Territory Manager Code

**Processing**:

- Used to lookup employee via `tmPositionId`
- **Validation**: Records skipped if employee not found (`employee.id === 0`)
- **Mapping**: `tmPositionCode: +employee.tmPositionId`
- **Purpose**: Links RCPA data to responsible territory managers

### 6. `DOC_GSPED_FLAG` → GSP Compliance Flag

**Processing**:

- **Conversion**: `record?.DOC_GSPED_FLAG === 1` → boolean `gspFlag`
- **Purpose**: Tracks Good Sales Practice compliance for doctors
- **Compliance**: Used for regulatory reporting and doctor classification

### 7. `PRODUCT_SEQ` → Product Priority Sequence

**Processing**:

- **Lookup**: `this.priorities[record?.PRODUCT_SEQ?.trim()] || -1`
- **Default**: `-1` for unmapped or null sequences
- **Configuration**: Uses `settings.brandPrioritiesMap` from app config
- **Purpose**: Product priority classification for marketing analytics

### 8. `QUADRANTS_CY` → Performance Quadrant

**Processing**:

- **Lookup**: `this.quadrants[record?.QUADRANTS_CY] || -1`
- **Default**: `-1` for unmapped or null quadrants
- **Configuration**: Uses `settings.brandQuadrantsMap` from app config
- **Purpose**: Brand performance quadrant analysis

### 9. `BRAND_NAME` → Brand Identification

**Processing**:

- **Name Correction**: Uses `WordErrosMap.getBrandName(name)` for standardization
- **Division Validation**: Brand must exist in the same division as employee
- **Caching**: Brand lookups cached for 24 hours (86400 seconds)
- **Purpose**: Links RCPA data to specific pharmaceutical brands

## Data Processing Flow

### Division-Based Processing

```sql
-- First query: Count records by division
SELECT `EMP_DIVISION` as division, count(*) AS count
FROM `VW_TABLE_CON_DOCTOR_LIST`
WHERE (
  `DRL_RCPA_VALUE` IS NOT NULL
  OR `DRL_RCPA_QTY` IS NOT NULL
  OR `PRODUCT_SEQ` IS NOT NULL
  OR `QUADRANTS_CY` IS NOT NULL
)
AND `TIME_KEY` = '${timeKey}'
GROUP BY `EMP_DIVISION`
```

### Record Processing per Division

```sql
-- Second query: Fetch detailed records by division
SELECT
  `TIME_KEY`, `CEGEDIM_CUSTOMER_ID`, `EMP_DIVISION`,
  `DRL_RCPA_VALUE`, `DRL_RCPA_QTY`, `EMP_POSITION_CD`,
  `DOC_GSPED_FLAG`, `PRODUCT_SEQ`, `QUADRANTS_CY`, `BRAND_NAME`
FROM `VW_TABLE_CON_DOCTOR_LIST`
WHERE (
  `DRL_RCPA_VALUE` IS NOT NULL
  OR `DRL_RCPA_QTY` IS NOT NULL
  OR `PRODUCT_SEQ` IS NOT NULL
  OR `QUADRANTS_CY` IS NOT NULL
)
AND `TIME_KEY` = '${timeKey}'
AND `EMP_DIVISION` = '${division}'
```

### Record Transformation Logic

```typescript
const row: DoctorDataRcpaSeeder$Model = {
  tmPositionCode: +employee.tmPositionId,
  doctorId: doctor.id,
  rcpaQty,
  rcpa,
  brandId: brand.id,
  qtr,
  year: +timeKey.split('-')[0],
  gspFlag: record?.DOC_GSPED_FLAG === 1,
  productSeq: record?.PRODUCT_SEQ
    ? this.priorities[record?.PRODUCT_SEQ?.trim()] || -1
    : -1,
  quadrantMedian: record?.QUADRANTS_CY
    ? this.quadrants[record?.QUADRANTS_CY] || -1
    : -1,
};
```

### Data Validation Rules

```typescript
// Skip records if critical lookups fail
if (division.id === 0) continue;
if (employee.id === 0) continue;
if (doctor.id === 0) continue;

// Skip records with no meaningful data
if (
  rcpa === null &&
  rcpaQty === null &&
  row.productSeq === -1 &&
  row.quadrantMedian === -1
)
  continue;
```

## Database Schema

### Doctor Data RCPA Seeder Table

```sql
doctor_data_rcpa_seeder:
  - id (primary key)
  - tmPositionCode (territory manager position)
  - doctorId (foreign key to doctors)
  - rcpaQty (RCPA quantity value)
  - rcpa (RCPA monetary value)
  - brandId (foreign key to brands)
  - qtr (quarter: 1-4)
  - year (calendar year)
  - gspFlag (GSP compliance boolean)
  - productSeq (product priority sequence)
  - quadrantMedian (performance quadrant)
  - createdAt (record creation)
  - updatedAt (record update)
```

## Where Data is Consumed

### RCPA Analytics Dashboard

- **Doctor Performance**: Individual doctor RCPA tracking and trends
- **Territory Analysis**: Territory manager performance metrics
- **Brand Performance**: Brand-specific RCPA analytics and comparisons
- **Quarterly Reports**: Time-based RCPA trend analysis

### Sales Intelligence

- **Doctor Segmentation**: GSP compliance and performance quadrant analysis
- **Product Priority**: Priority-based product performance tracking
- **Revenue Analytics**: RCPA value aggregation and analysis
- **Volume Metrics**: Quantity-based prescription tracking

### Compliance Monitoring

- **GSP Compliance**: Good Sales Practice flag monitoring
- **Regulatory Reporting**: Compliance-based doctor classification
- **Audit Trails**: RCPA data audit and verification
- **Performance Standards**: Doctor performance against compliance standards

### Business Intelligence

- **Market Share Analysis**: Brand performance and market position
- **Doctor Engagement**: Healthcare provider engagement metrics
- **Territory Optimization**: Territory performance and optimization
- **Strategic Planning**: Data-driven sales strategy development

### Caching Strategy

```typescript
// Brand lookup with 24-hour cache
await CacheStore().remember<Brand$Model>(
  `brand: name[${name}]: division[${divisionId}]`,
  async () => {
    /* brand lookup logic */
  },
  86400, // 24 hours
);
```

### Bulk Operations

```typescript
// Bulk insert every 5000 records
if (rows.length >= 5000) {
  await this.doctorLib.doctorDataRcpaSeederRepo.bulkInsert(rows);
  rows = [];
}
```

## Data Examples

### Sample BigQuery Record

```typescript
const record = {
  TIME_KEY: { value: '2024-04-30' },
  CEGEDIM_CUSTOMER_ID: 3559322,
  EMP_DIVISION: 'Maximus',
  DRL_RCPA_VALUE: 15000.5,
  DRL_RCPA_QTY: 250,
  EMP_POSITION_CD: 50134380,
  DOC_GSPED_FLAG: 1,
  PRODUCT_SEQ: 'P001',
  QUADRANTS_CY: 'Q1',
  BRAND_NAME: 'Telsartan-M',
};
```

### Processed Database Record

```sql
INSERT INTO doctor_data_rcpa_seeder (
  tmPositionCode, doctorId, rcpaQty, rcpa, brandId,
  qtr, year, gspFlag, productSeq, quadrantMedian
) VALUES (
  50134380, 12345, 250, 15000.50, 678,
  2, 2024, true, 1, 1
);
```

### Configuration Examples

```typescript
// Brand Priorities Map
settings.brandPrioritiesMap = {
  P001: 1,
  P002: 2,
  P003: 3,
};

// Brand Quadrants Map
settings.brandQuadrantsMap = {
  Q1: 1,
  Q2: 2,
  Q3: 3,
  Q4: 4,
};
```
