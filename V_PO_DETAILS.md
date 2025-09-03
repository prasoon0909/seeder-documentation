# BigQuery V_PO_DETAILS Column Usage Mapping

## BigQuery Source Columns Used

### From `V_PO_DETAILS` table in BigQuery:

| BigQuery Column | Maps To          | Used In                   | Purpose                                           |
| --------------- | ---------------- | ------------------------- | ------------------------------------------------- |
| `PO_NUMBER`     | `po.poNumber`    | Purchase Order Management | Unique purchase order identifier                  |
| `DESCRIPTION`   | `po.description` | Purchase Order Details    | Purchase order description and details            |
| `CREATED_DATE`  | `po.createdDate` | Purchase Order Tracking   | Purchase order creation timestamp                 |
| `PO_STATUS`     | Not stored       | Query only                | Purchase order status (referenced but not stored) |

## Column Usage Details

### 1. `PO_NUMBER` → `poNumber`

**Usage**:

- Primary key for purchase order records
- Used in createOrUpdate operations to prevent duplicates
- Links to promotional activities and sample distribution

### 2. `DESCRIPTION` → `description`

**Processing**:

- **HTML Tag Removal**: Strips HTML tags using regex `(<([^>]+)>)`
- **Entity Decoding**: Converts HTML entities to readable text
  - `&nbsp;` → space
  - `&amp;` → `&`
  - `&ndash;` → space
  - `&mdash;` → space
- **Space Normalization**: Complex space cleanup process
- **Length Limitation**: Truncated to 255 characters
- **Final Result**: Clean, readable purchase order description

### 3. `CREATED_DATE` → `createdDate`

**Processing**:

- Extracts date from BigQueryTimestamp object
- Converts to JavaScript Date object
- Handles null values gracefully
- Used for purchase order timeline tracking

### 4. `PO_STATUS`

**Usage**:

- Retrieved in query but not stored in local database
- Available for potential future processing
- Example values: 'N' (New), other status codes

## Data Processing Flow

### Description Cleaning Process

```typescript
// Multi-step text cleaning process
const description = record?.DESCRIPTION?.replace(/(<([^>]+)>)/gi, ' ') // Remove HTML tags
  .replaceAll('&nbsp;', ' ') // Convert non-breaking spaces
  .replaceAll('&amp;', '&') // Convert ampersand entities
  .replaceAll('&ndash;', ' ') // Convert en-dash entities
  .replaceAll('&mdash;', ' ') // Convert em-dash entities
  .replaceAll(' ', '<>') // Temporary space replacement
  .replaceAll('><', '') // Remove empty tag pairs
  .replaceAll('<>', ' ') // Restore spaces
  .trim() // Remove leading/trailing spaces
  .substring(0, 255); // Limit to 255 characters
```

### Purchase Order Creation

```typescript
await this.poLib.repo.createOrUpdate(
  { poNumber: record.PO_NUMBER }, // Unique identifier
  {
    description, // Cleaned description
    createdDate: date, // Processed date
  },
);
```

## Database Schema

### Purchase Orders Table (po)

```sql
po:
  - id (primary key)
  - poNumber (unique purchase order number)
  - description (cleaned PO description, max 255 chars)
  - createdDate (PO creation timestamp)
  - createdAt (record creation)
  - updatedAt (record update)
```

## Where Data is Consumed

### Promotional Activities

- **Sample Distribution**: Links PO numbers to sample campaigns
- **Promotional Campaigns**: Associates purchase orders with marketing activities
- **Activity Tracking**: Connects activities to specific purchase orders
- **Financial Tracking**: Links promotional spend to purchase orders

### Marketing Analytics

- **Campaign Cost Analysis**: Purchase order costs for marketing campaigns
- **ROI Calculation**: Marketing spend vs. activity outcomes
- **Budget Tracking**: Promotional budget allocation and usage
- **Audit Trail**: Financial audit trail for marketing expenses

### Business Intelligence

- **Spend Analysis**: Purchase order spend patterns
- **Vendor Management**: Purchase order vendor analysis
- **Timeline Analysis**: Purchase order creation trends
- **Description Analytics**: Common purchase patterns and items

## Data Quality Requirements

### Required Fields

- **PO_NUMBER**: Must be unique, non-null purchase order identifier
- **DESCRIPTION**: Should contain meaningful purchase order details
- **CREATED_DATE**: Should be valid timestamp for timeline tracking

### Data Validation

- **Unique PO Numbers**: Prevents duplicate purchase order entries
- **Description Cleaning**: Ensures clean, readable descriptions
- **Date Handling**: Gracefully handles null or invalid dates
- **Length Limits**: Enforces 255-character limit on descriptions

### HTML Content Handling

- **Tag Removal**: Strips all HTML tags for clean text
- **Entity Conversion**: Converts HTML entities to readable characters
- **Space Normalization**: Removes extra spaces and formatting artifacts
- **Character Safety**: Ensures database-safe character storage

## Performance Considerations

### Processing Efficiency

- **Batch Processing**: Processes 1000 records at a time for monitoring
- **Upsert Operations**: Uses createOrUpdate to handle duplicates efficiently
- **String Processing**: Complex but necessary text cleaning operations
- **Date Conversion**: Efficient BigQueryTimestamp to Date conversion

### Memory Management

- **Streaming Processing**: Processes records one at a time
- **String Operations**: Multiple string operations per record
- **Garbage Collection**: Frequent temporary string creation

## Monitoring and Logging

### Progress Tracking

```typescript
// Logs progress every 1000 records
if (i % 1000 === 0) console.log('Count =>', i);

// Initial data summary
console.log('Po records:', records.length);
```

### Processing Metrics

- **Total Records**: Number of purchase orders in BigQuery
- **Processing Rate**: Records processed per batch
- **Completion Tracking**: Progress through entire dataset

## Data Examples

### Sample BigQuery Record

```typescript
const record = {
  PO_NUMBER: '5800686478',
  DESCRIPTION: 'Back Plate for EXP-120 CT, Shaft for EXP-120 CT pump, Pump Casing for EXP-120 CT pump, Impeller for EXP-120 CT pump',
  CREATED_DATE: BigQueryTimestamp { value: '2024-03-30T00:00:00.000Z' },
  PO_STATUS: 'N'
}
```

### Processed Database Record

```sql
INSERT INTO po (poNumber, description, createdDate) VALUES (
  '5800686478',
  'Back Plate for EXP-120 CT, Shaft for EXP-120 CT pump, Pump Casing for EXP-120 CT pump, Impeller for EXP-120 CT pump',
  '2024-03-30 00:00:00'
)
```
