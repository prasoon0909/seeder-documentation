# BigQuery VBRAND_DIM_SOG Column Usage Mapping

## BigQuery Source Columns Used

### From `VBRAND_DIM_SOG` table in BigQuery:

| BigQuery Column | Maps To | Used In | Purpose |
|-----------------|---------|---------|---------|
| `DIVISION_CODE` | `divisions.refId` | Division Management, Initiative Analytics | Pharmaceutical division identification |
| `DIVISION` | `divisions.name` | Division Management, SOG Dashboard | Division name and filtering |
| `BRAND_CODE` | Not directly stored | Brand reference | Brand identification (legacy) |
| `BRAND` | `brands.name` | Initiative Management, Brand Analytics | Brand name and marketing campaigns |
| `BRAND_CLASSIFICATION` | `brands.classification` | Brand Analytics, Reporting | Brand categorization (e.g., Milk, OTC) |

## Column Usage Details

### 1. `DIVISION_CODE` → `divisions.refId`
**Usage**: 
- Unique identifier for pharmaceutical divisions
- Links to initiatives and marketing campaigns
- Used in SOG Dashboard filtering

### 2. `DIVISION` → `divisions.name`
**Processing**: 
- Converted to lowercase and trimmed
- Used for division lookups and matching
- Key for linking brands to divisions

### 3. `BRAND_CODE`
**Usage**:
- Legacy brand identification
- Not stored directly in local database
- Used during seeding process only

### 4. `BRAND` → `brands.name`
**Processing**:
- Cleaned using WordErrosMap.getBrandName()
- Converts to lowercase
- Links to marketing initiatives and campaigns

### 5. `BRAND_CLASSIFICATION` → `brands.classification`
**Usage**:
- Optional field for brand categorization
- Converted to lowercase
- Used in brand analytics and reporting

## Data Processing Flow

### Division Processing
```typescript
// Create or update divisions
const code = division.DIVISION_CODE;
const nameLower = division?.DIVISION?.toLowerCase()?.trim();
await repo.createOrUpdate({ refId: code, name: nameLower }, {});
```

### Brand Processing
```typescript
// Link brands to divisions and process names
const name = WordErrosMap.getBrandName(brand?.BRAND?.toLowerCase());
const classification = brand.BRAND_CLASSIFICATION?.toLowerCase() || undefined;
const updateObj = { name, divisionId: division.id };
await repo.createOrUpdate(updateObj, { classification });
```

## Where Data is Consumed

### Division Usage
- **Initiative Management**: Links initiatives to pharmaceutical divisions
- **SOG Dashboard**: Division-based filtering and analytics
- **Employee Management**: Assigns staff to divisions
- **Doctor Management**: Links healthcare providers to divisions

### Brand Usage
- **Initiative Creation**: Assigns brands to marketing campaigns
- **Brand Analytics**: Performance tracking by brand
- **Classification Reports**: Brand category analysis
- **Marketing ROI**: Brand-specific return calculations

## Data Quality Requirements

- **DIVISION_CODE**: Must be numeric, unique identifier
- **DIVISION**: Must not be null or empty after trimming
- **BRAND**: Required for brand creation, processed through WordErrosMap
- **BRAND_CLASSIFICATION**: Optional, converted to lowercase if present
- **Division-Brand Relationship**: Each brand must link to valid division
