# BigQuery V_USER_DIM Column Usage Mapping

## BigQuery Source Columns Used

### From `V_USER_DIM` table in BigQuery:

| BigQuery Column     | Maps To                     | Used In               | Purpose                                 |
| ------------------- | --------------------------- | --------------------- | --------------------------------------- |
| `PMT_ID`            | `users.refId`               | User Management       | Unique user identifier                  |
| `PMT_NAME`          | `users.name`                | User Management       | User display name                       |
| `DIVISION_NAME`     | `user_divisions.divisionId` | User-Division Linking | Links users to pharmaceutical divisions |
| `PMT_EMAIL_ADDRESS` | `users.email`               | User Authentication   | User login email and communication      |

## Column Usage Details

### 1. `PMT_ID` → `refId`

**Usage**:

- Unique identifier for users in external PMT system
- Used as reference ID for user records
- Links to external system integration

### 2. `PMT_NAME` → `name`

**Processing**:

- Trimmed to remove leading/trailing whitespace
- Used as display name in user interface
- Example: "Neeraj Tewari Tewari" becomes user display name

### 3. `DIVISION_NAME` → Division Linking

**Processing**:

- **Special Case**: When `DIVISION_NAME = '-1'`, user gets access to ALL divisions
- **Normal Case**: Links user to specific pharmaceutical division
- **Conversion**: Converted to lowercase for division matching
- **Multi-Division Support**: Users can be linked to multiple divisions

### 4. `PMT_EMAIL_ADDRESS` → `email`

**Processing**:

- Converted to lowercase for consistency
- Used as unique login identifier
- **Special Case Handling**: Specific email transformations for MS login compatibility
- Primary key for user identification and authentication

## Data Processing Flow

### Division Assignment Logic

```typescript
let divisionIds = [];
if (record.DIVISION_NAME !== '-1') {
  // Assign specific division
  const divisionName = record?.DIVISION_NAME?.toLowerCase();
  const division = divisions[divisionName]
    ? divisions[divisionName][0]
    : { id: 0 };
  divisionIds.push(division.id);
} else {
  // Assign ALL divisions (super user access)
  divisionIds = [...Object.values(divisions).map((s) => s[0].id)];
}
```

### User Creation/Update Logic

```typescript
// Check if user exists by email
let user = await this.lib.repo.firstWhere({ email: pmtEmailAddress }, false);

if (user) {
  // Update existing user
  user = await this.lib.repo.updateAndReturn(
    { id: user.id },
    {
      name: record.PMT_NAME?.trim(),
      email: record.PMT_EMAIL_ADDRESS.toLowerCase(),
      mobileNumber: null,
    },
  );
} else {
  // Create new user
  user = await this.lib.repo.create({
    uuid: v4(),
    refId: record.PMT_ID,
    name: record.PMT_NAME?.trim(),
    email: record.PMT_EMAIL_ADDRESS.toLowerCase(),
    mobileNumber: null,
    status: AppConfig.get('settings.users.status.active'),
  });
}

// Attach user to divisions
await this.lib.repo.attach(user, 'divisions', divisionIds);
```

### Special Email Handling

```typescript
// Handle unique cases for users to support MS login
if (record.PMT_EMAIL_ADDRESS === 'aniruddhachakraborty@drreddys.com') {
  record.PMT_EMAIL_ADDRESS = 'p90008357@dev.mydrreddys.com';
}
```

## Database Schema

### Users Table

```sql
users:
  - id (primary key)
  - uuid (unique identifier)
  - refId (PMT system ID)
  - name (user display name)
  - email (login email, unique)
  - mobileNumber (phone number, nullable)
  - password (hashed password)
  - status (user status, active/inactive)
  - createdAt (record creation)
  - updatedAt (record update)
```

### User-Division Relationship

```sql
user_divisions:
  - user_id (foreign key to users)
  - division_id (foreign key to divisions)
  - created_at (relationship creation)
  - updated_at (relationship update)
```

## Where Data is Consumed

### User Authentication

- **Login System**: Email-based user authentication
- **Access Control**: Division-based access permissions
- **Session Management**: User session and authorization
- **Role Management**: User role assignments and permissions

### User Management

- **User Directory**: Comprehensive user information system
- **Profile Management**: User profile data and preferences
- **Division Access**: Multi-division access control
- **User Administration**: User lifecycle management

### Business Intelligence

- **User Analytics**: User activity and engagement tracking
- **Access Patterns**: Division access and usage patterns
- **User Segmentation**: User categorization and analysis
- **Security Auditing**: User access and security monitoring

## Data Quality Requirements

### Required Fields

- **PMT_ID**: Unique identifier from PMT system
- **PMT_NAME**: User display name (trimmed)
- **PMT_EMAIL_ADDRESS**: Valid email for authentication
- **DIVISION_NAME**: Division assignment (including `-1` for all divisions)

### Data Validation

- **Email Uniqueness**: Emails must be unique across system
- **Division Validation**: Division names must exist in divisions table
- **Special Cases**: Handles specific email transformations
- **Default Values**: Sets default password and active status

### Data Processing Rules

- **Upsert Logic**: Updates existing users or creates new ones
- **Email Normalization**: Converts emails to lowercase
- **Name Trimming**: Removes whitespace from names
- **Division Replacement**: Replaces existing division assignments

## Data Examples

### Sample BigQuery Record

```typescript
const record = {
  PMT_ID: '64048',
  PMT_NAME: 'Neeraj Tewari Tewari',
  DIVISION_NAME: 'Maximus',
  PMT_EMAIL_ADDRESS: 'neeraj.tewari@drreddys.com',
};
```

### Processed Database Records

```sql
-- Users table
INSERT INTO users (uuid, refId, name, email, password, status) VALUES (
  'generated-uuid',
  '64048',
  'Neeraj Tewari Tewari',
  'neeraj.tewari@drreddys.com',
  'hashed-password',
  1
);

-- User-Division relationship
INSERT INTO user_divisions (user_id, division_id) VALUES (
  user_id_for_neeraj, division_id_for_maximus
);
```

### Super User Example

```typescript
const superUserRecord = {
  PMT_ID: '12345',
  PMT_NAME: 'System Administrator',
  DIVISION_NAME: '-1', // Gets access to ALL divisions
  PMT_EMAIL_ADDRESS: 'admin@drreddys.com',
};
```
