# BigQuery V_TOWN_POP_EMP_DOC_DATA Column Usage Mapping

## BigQuery Source Columns Used

### From `V_TOWN_POP_EMP_DOC_DATA` table in BigQuery:

| BigQuery Column | Maps To | Used In | Purpose |
|-----------------|---------|---------|---------|
| `EMP_POSITION_CD` | `doctor_data.tmPositionId` | Employee Matching | Employee position identification |
| `CEGEDIM_CUSTOMER_ID` | `doctor_data.doctorId` | Doctor Matching | Healthcare provider identification |
| `POP_CATEGORY` | `doctor_data.cityTier` | Population Analytics | City tier/population category classification |

## Column Usage Details

### 1. `EMP_POSITION_CD` → Employee Matching
**Usage**: 
- Links to employee records via tmPositionId
- Required for identifying correct doctor-employee combinations
- Used in conjunction with doctorId and divisionId for precise matching

### 2. `CEGEDIM_CUSTOMER_ID` → Doctor Matching
**Usage**:
- Primary healthcare provider identifier
- Links to doctors table via refId
- Essential for updating correct doctor records

### 3. `POP_CATEGORY` → `cityTier`
**Processing**:
- Trimmed for whitespace removal
- Mapped through `settings.populationMap` configuration
- Updates existing doctor_data records with population classification
- Defaults to -1 for unmapped categories

## Data Processing Flow

### Population Update Process
```typescript
// Updates existing doctor_data records with population information
await this.doctorLib.doctorDataRepo.updateWhere(
  {
    doctorId: doctor.id,
    tmPositionId: employee.tmPositionId,
    divisionId: employee.divisionId,
  },
  { cityTier: populationType || -1 }
);
```

### Population Mapping
```typescript
// Maps BigQuery categories to internal tier system
const populationMap = AppConfig.get('settings.populationMap');
const populationType = populationMap[record.POP_CATEGORY?.trim()];
```

## Database Updates

### Target Table: doctor_data
**Updated Field**: `cityTier`
**Update Logic**: 
- Matches on doctorId + tmPositionId + divisionId
- Updates cityTier field with mapped population category
- Preserves all other existing data

### Matching Criteria
```sql
WHERE doctorId = matched_doctor_id
  AND tmPositionId = employee_position_id  
  AND divisionId = employee_division_id
```

## Where Data is Consumed

### Population Analytics
- **City Tier Analysis**: Urban vs rural doctor distribution
- **Market Segmentation**: Population-based targeting strategies
- **Resource Allocation**: Territory planning based on population density
- **Performance Analytics**: ROI analysis by city tier

### Territory Management
- **Coverage Planning**: Population-weighted territory assignments
- **Market Potential**: City tier impact on prescription volumes
- **Strategic Planning**: Investment decisions by population category

### Business Intelligence
- **Demographic Reports**: Doctor distribution by population tiers
- **Market Analysis**: Prescription patterns by city classification
- **ROI Calculations**: Marketing spend efficiency by population density

## Configuration Requirements

### Population Mapping (`settings.populationMap`)
```typescript
// Example configuration structure
populationMap: {
  "TIER_1": 1,     // Metro cities
  "TIER_2": 2,     // Large cities  
  "TIER_3": 3,     // Medium cities
  "TIER_4": 4,     // Small towns
  "RURAL": 5       // Rural areas
}
```

## Data Quality Requirements

### Required Fields
- **EMP_POSITION_CD**: Must exist in employees table
- **CEGEDIM_CUSTOMER_ID**: Must exist in doctors table
- **POP_CATEGORY**: Must not be NULL (filtered in query)

### Validation Logic
- **Employee Validation**: Skips records with invalid employee positions
- **Doctor Validation**: Skips records with non-existent doctors
- **Population Validation**: Uses -1 for unmapped population categories

### Data Consistency
- **Existing Records**: Only updates existing doctor_data combinations
- **Preservation**: Maintains all other doctor_data fields unchanged
- **Batch Processing**: Processes 1000 records at a time for monitoring

### Success Metrics
- **Updated Count**: Number of successful doctor_data updates
- **Skipped Count**: Records with invalid doctor or employee references
- **Coverage Rate**: Percentage of records successfully processed

## Error Handling

### Skip Conditions
1. **Invalid Doctor**: CEGEDIM_CUSTOMER_ID not found in doctors table
2. **Invalid Employee**: EMP_POSITION_CD not found in employees table
3. **NULL Population**: POP_CATEGORY is NULL (filtered in query)

### Fallback Behavior
- **Unmapped Categories**: Sets cityTier to -1 for unknown POP_CATEGORY values
- **Cache Misses**: Returns {id: 0} for non-existent doctors
- **Continue Processing**: Skips invalid records but continues with remaining data

## Usage in Analytics

### City Tier Reports
- Doctor distribution across population tiers
- Prescription volume correlation with city tiers
- Market penetration by population density

### Strategic Planning
- Resource allocation based on population categories
- Marketing budget distribution by city tiers
- Territory optimization using population data

### Performance Metrics
- ROI analysis by city tier
- Engagement rates across population categories
- Market share analysis by urban/rural classification
