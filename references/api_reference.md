# PocketBase API Reference

Quick reference for common PocketBase operations in hooks.

## DAO (Database Access Object)

### Finding Records

```javascript
// Find by ID
const record = $app.dao().findRecordById("collectionName", "recordId")

// Find first matching record
const record = $app.dao().findFirstRecordByData("users", "email", "user@example.com")

// Find with filter
const record = $app.dao().findFirstRecordByFilter(
    "users",
    "email = {:email} && active = true",
    {"email": "user@example.com"}
)

// Find multiple records
const records = $app.dao().findRecordsByFilter(
    "posts",
    "status = 'published' && author = {:authorId}",
    "-created",  // sort order (- for DESC, + for ASC)
    50,          // limit
    {"authorId": "xyz123"}
)

// Find all records (no filter)
const records = $app.dao().findRecordsByFilter("posts", "1=1", "-created", 100)
```

### Saving Records

```javascript
// Save with hooks
$app.dao().saveRecord(record)

// Save without triggering hooks
$app.dao().withoutHooks().saveRecord(record)

// Save without validation
$app.saveNoValidate(record)

// Expand relations when saving
$app.dao().expandRecord(record, ["author", "comments"], null)
```

### Deleting Records

```javascript
// Delete with hooks
$app.dao().deleteRecord(record)

// Delete without hooks
$app.dao().withoutHooks().deleteRecord(record)

// Delete file from record
$app.dao().deleteFileById(record, "filename.jpg")
```

### Collections

```javascript
// Get collection
const collection = $app.dao().findCollectionByNameOrId("posts")

// Create new record instance
const record = new Record(collection)
```

## Record Operations

### Getting/Setting Values

```javascript
// Get value
const value = record.get("fieldName")

// Set value
record.set("fieldName", value)

// Get original (before changes)
const original = record.originalCopy()

// Get clean copy
const clean = record.cleanCopy()

// Get ID
const id = record.id

// Get collection name
const collectionName = record.collection().name
```

### Working with Auth Records

```javascript
// Get email
const email = record.email()

// Check if email is verified
const isVerified = record.verified()

// Get username (if collection has username field)
const username = record.username()
```

### Hiding/Showing Fields

```javascript
// Hide field from response
record.hide("secretField")

// Hide multiple fields
record.hide("field1", "field2", "field3")

// Enable custom data (required for computed fields)
record.withCustomData(true)
```

### File Operations

```javascript
// Get file URL
const fileUrl = record.fileUrl("avatar", "thumbnail")

// Get all file URLs for multi-file field
const fileUrls = record.fileUrls("attachments")
```

## HTTP Operations

### Making HTTP Requests

```javascript
// GET request
const response = $http.send({
    url: "https://api.example.com/data",
    method: "GET",
    headers: {
        "Authorization": "Bearer token123"
    }
})

// POST request
const response = $http.send({
    url: "https://api.example.com/data",
    method: "POST",
    body: JSON.stringify({key: "value"}),
    headers: {
        "Content-Type": "application/json"
    }
})

// Access response
console.log(response.statusCode)
console.log(response.headers)
const data = response.json  // Parsed JSON
const text = response.raw    // Raw response
```

## Email Operations

### Sending Email

```javascript
const message = new MailerMessage({
    from: {
        address: $app.settings().meta.senderAddress,
        name: $app.settings().meta.senderName,
    },
    to: [
        {address: "user@example.com", name: "User Name"}
    ],
    cc: [
        {address: "cc@example.com"}
    ],
    bcc: [
        {address: "bcc@example.com"}
    ],
    subject: "Email Subject",
    html: "<h1>HTML Content</h1>",
    text: "Plain text content",
    headers: {
        "X-Custom-Header": "value"
    }
})

$app.newMailClient().send(message)
```

## Settings

### Accessing Settings

```javascript
// Get all settings
const settings = $app.settings()

// Get meta settings
const meta = $app.settings().meta

// Sender email
const senderEmail = $app.settings().meta.senderAddress
const senderName = $app.settings().meta.senderName

// App name and URL
const appName = $app.settings().meta.appName
const appUrl = $app.settings().meta.appUrl

// Logs settings
const maxDays = $app.settings().logs.maxDays
```

## Validation

### Validating Records

```javascript
// Validate record
$app.validate(record)

// Validate without throwing (returns validation errors)
try {
    $app.validate(record)
} catch (err) {
    console.log("Validation errors:", err.data)
}
```

## Realtime

### Sending Realtime Messages

```javascript
// Send message to specific subscription
$app.realtimeServer().send(
    "collectionName",  // topic
    {
        action: "create",
        record: record
    }
)
```

## Utilities

### Token Generation

```javascript
// Generate record token
const token = $tokens.recordAuthToken($app, record)

// Generate admin token  
const token = $tokens.adminAuthToken($app, admin)
```

### Security

```javascript
// Hash password
const hash = $security.hashPassword("password123")

// Compare password with hash
const matches = $security.compareHashAndPassword(hash, "password123")

// Generate random string
const randomStr = $security.randomString(32)

// Generate random string with alphabet
const randomStr = $security.randomStringWithAlphabet(32, "0123456789abcdef")
```

### Date/Time

```javascript
// Current date/time in ISO format
const now = new Date().toISOString()

// Parse date string
const date = new Date("2024-01-15T10:30:00Z")

// Format for PocketBase filters (RFC3339)
const formatted = date.toISOString()  // "2024-01-15T10:30:00.000Z"
```

## Request Context (in Request hooks)

### Accessing Request Data

```javascript
// Get HTTP request
const req = e.httpContext.request()

// Get auth record
const authRecord = e.httpContext.get("authRecord")

// Get query parameter
const page = e.httpContext.queryParam("page")

// Get form value
const title = e.httpContext.formValue("title")

// Get header
const userAgent = req.header.get("User-Agent")

// Get client IP
const ip = e.httpContext.realIP()
```

## Filter Syntax

### Query Parameters

```javascript
// Basic equality
"name = 'John'"

// Comparison
"age >= 18"
"created > '2024-01-01 00:00:00.000Z'"

// Multiple conditions
"status = 'active' && age >= 18"
"role = 'admin' || role = 'moderator'"

// LIKE/Contains
"name ~ 'john'"        // Case-insensitive contains
"email !~ '@test'"     // Does not contain

// Array fields
"tags ?= 'javascript'"  // Array contains value
"roles ?!= 'admin'"     // Array doesn't contain value

// Relation fields
"author.name = 'John'"
"team.status = 'active'"

// Check for empty/null
"description = ''"
"optionalField = null"

// Parameterized (recommended)
"email = {:email} && status = {:status}"
```

### Sort Order

```javascript
"-created"           // Descending by created
"+name"              // Ascending by name
"-created,+name"     // Multiple fields
```

## Common Patterns

### Checking if Record Exists

```javascript
try {
    const record = $app.dao().findRecordById("users", userId)
    // Record exists
} catch (err) {
    // Record doesn't exist
}
```

### Safe Record Access

```javascript
try {
    const author = $app.dao().findRecordById("users", record.get("author"))
    // Use author
} catch (err) {
    console.error("Author not found:", err)
    // Handle missing author
}
```

### Updating Multiple Records

```javascript
const records = $app.dao().findRecordsByFilter(
    "tasks",
    "project = {:projectId}",
    "",
    0,
    {"projectId": projectId}
)

records.forEach(record => {
    record.set("status", "completed")
    $app.dao().withoutHooks().saveRecord(record)
})
```

### Transaction-like Operations

```javascript
// While there's no explicit transaction API in hooks,
// operations within a single hook before e.next() are part
// of the same database transaction

onRecordCreate((e) => {
    // These all happen in the same transaction
    e.record.set("status", "pending")

    // Create related record
    const relatedRecord = new Record($app.dao().findCollectionByNameOrId("related"))
    relatedRecord.set("parentId", e.record.id)
    $app.dao().saveRecord(relatedRecord)

    e.next()  // Transaction commits after this
}, "main_collection")
```

## Backup & Recovery

### Manual Backup

The entire PocketBase application state is contained in the `pb_data` directory.

```bash
# Stop PocketBase for transactional safety (optional but recommended)
systemctl stop pocketbase  # or kill the process

# Backup the entire pb_data directory
cp -r pb_data pb_data_backup_$(date +%Y%m%d_%H%M%S)

# Or create compressed backup
tar -czf pb_data_backup_$(date +%Y%m%d_%H%M%S).tar.gz pb_data

# Restart PocketBase
systemctl start pocketbase
```

### Automated Backups via Hooks

```javascript
/// <reference path="../pb_data/types.d.ts" />

// Customize what gets backed up
onBackupCreate((e) => {
    // Exclude unwanted directories (added in v0.30+)
    e.exclude.push("lost+found")
    e.exclude.push("temp")

    console.log("Creating backup:", e.name)

    e.next()
})

// Post-backup actions
onBackupRestore((e) => {
    console.log("Restoring from backup:", e.name)

    e.next()
})
```

### Restore from Backup

```bash
# Stop PocketBase
systemctl stop pocketbase

# Backup current state (safety)
mv pb_data pb_data_old

# Restore from backup
tar -xzf pb_data_backup_YYYYMMDD_HHMMSS.tar.gz
# or: cp -r pb_data_backup_YYYYMMDD_HHMMSS pb_data

# Start PocketBase
systemctl start pocketbase

# Verify restoration
curl http://localhost:8090/api/health
```

### What Gets Backed Up

The `pb_data` directory contains:
- `data.db` - Main SQLite database (all records, collections, settings)
- `logs.db` - Request logs (if enabled)
- `storage/` - Uploaded files
- `backups/` - Automatic backups (if configured)

### Production Best Practices

1. **Backup Schedule**: Daily backups minimum for production
2. **Transactional Safety**: Stop PocketBase or use backup hooks
3. **Offsite Storage**: Copy backups to separate server/cloud storage
4. **Test Restores**: Regularly verify backup restoration works
5. **Retention Policy**: Keep multiple backup generations (e.g., 30 days)
6. **Monitor Backups**: Alert on backup failures

### Backup Script Example

```bash
#!/bin/bash
# save as: /usr/local/bin/backup-pocketbase.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/pocketbase"
PB_DATA="/opt/pocketbase/pb_data"
RETENTION_DAYS=30

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Create backup (PocketBase keeps running)
tar -czf "$BACKUP_DIR/pb_data_$DATE.tar.gz" -C "$(dirname "$PB_DATA")" "$(basename "$PB_DATA")"

# Verify backup was created
if [ -f "$BACKUP_DIR/pb_data_$DATE.tar.gz" ]; then
    echo "Backup created: pb_data_$DATE.tar.gz"

    # Copy to remote server (optional)
    # rsync -az "$BACKUP_DIR/pb_data_$DATE.tar.gz" user@backup-server:/backups/

    # Remove old backups
    find "$BACKUP_DIR" -name "pb_data_*.tar.gz" -mtime +$RETENTION_DAYS -delete
else
    echo "ERROR: Backup failed!"
    exit 1
fi
```

Add to crontab for daily backups:
```bash
# Run daily at 2 AM
0 2 * * * /usr/local/bin/backup-pocketbase.sh >> /var/log/pocketbase-backup.log 2>&1
```

### Important Notes

- PocketBase can run during backups, but stopping ensures consistency
- Settings encryption cannot be disabled once enabled
- Backups include all collections, settings, files, and logs
- Database size grows over time - monitor disk space
- Backup before major updates or migrations
