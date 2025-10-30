# PocketBase Hook Patterns

Practical examples of common hook implementations.

## Table of Contents
1. [Auto-Population](#auto-population)
2. [Validation](#validation)
3. [Notifications](#notifications)
4. [Cascading Updates](#cascading-updates)
5. [Soft Deletes](#soft-deletes)
6. [Audit Logging](#audit-logging)
7. [File Management](#file-management)

## Auto-Population

### Set Creation Timestamp
```javascript
/// <reference path="../pb_data/types.d.ts" />

onRecordCreate((e) => {
    e.record.set("createdAt", new Date().toISOString())
    e.next()
}, "posts")
```

### Auto-Set Owner
```javascript
onRecordCreateRequest((e) => {
    // Ensure record is created by authenticated user
    e.record.set("owner", e.httpContext.get("authRecord").id)
    e.next()
}, "documents")
```

### Generate Slug from Title
```javascript
onRecordCreate((e) => {
    const title = e.record.get("title")
    const slug = title
        .toLowerCase()
        .replace(/[^a-z0-9]+/g, '-')
        .replace(/^-|-$/g, '')
    
    e.record.set("slug", slug)
    e.next()
}, "articles")
```

### Set Sequential Number
```javascript
onRecordCreateExecute((e) => {
    // Get max order number
    const records = $app.dao().findRecordsByFilter(
        "orders",
        "1=1",
        "-orderNumber",
        1
    )
    
    const lastNumber = records.length > 0 ? records[0].get("orderNumber") : 0
    e.record.set("orderNumber", lastNumber + 1)
    
    e.next()
}, "orders")
```

## Validation

### Prevent Field Modification
```javascript
onRecordUpdate((e) => {
    const original = e.record.originalCopy()
    
    // Prevent changing verified status
    if (e.record.get("verified") !== original.get("verified")) {
        throw new Error("Cannot modify verified status")
    }
    
    e.next()
}, "users")
```

### Validate Related Records Exist
```javascript
onRecordCreate((e) => {
    const teamId = e.record.get("team")
    
    try {
        $app.dao().findRecordById("teams", teamId)
    } catch (err) {
        throw new Error("Invalid team specified")
    }
    
    e.next()
}, "projects")
```

### Business Logic Validation
```javascript
onRecordValidate((e) => {
    const startDate = new Date(e.record.get("startDate"))
    const endDate = new Date(e.record.get("endDate"))
    
    if (endDate <= startDate) {
        throw new Error("End date must be after start date")
    }
    
    // Check for overlapping bookings
    const overlapping = $app.dao().findFirstRecordByFilter(
        "bookings",
        `resource = {:resource} && id != {:id} && ((startDate <= {:endDate} && endDate >= {:startDate}))`,
        {
            "resource": e.record.get("resource"),
            "id": e.record.id || "",
            "startDate": e.record.get("startDate"),
            "endDate": e.record.get("endDate")
        }
    )
    
    if (overlapping) {
        throw new Error("Resource is already booked for this time period")
    }
    
    e.next()
}, "bookings")
```

### Require Unique Field Combination
```javascript
onRecordValidate((e) => {
    const username = e.record.get("username")
    const domain = e.record.get("domain")
    
    const existing = $app.dao().findFirstRecordByFilter(
        "users",
        `username = {:username} && domain = {:domain} && id != {:id}`,
        {
            "username": username,
            "domain": domain,
            "id": e.record.id || ""
        }
    )
    
    if (existing) {
        throw new Error("Username already exists in this domain")
    }
    
    e.next()
}, "users")
```

## Notifications

### Send Email on Record Create
```javascript
onRecordAfterCreateSuccess((e) => {
    const user = $app.dao().findRecordById("users", e.record.get("user"))
    
    const message = new MailerMessage({
        from: {
            address: $app.settings().meta.senderAddress,
            name: $app.settings().meta.senderName,
        },
        to: [{address: user.email()}],
        subject: "New post published",
        html: `<p>Your post "${e.record.get("title")}" has been published!</p>`
    })
    
    $app.newMailClient().send(message)
    
    e.next()
}, "posts")
```

### Notify on Status Change
```javascript
onRecordUpdate((e) => {
    const original = e.record.originalCopy()
    const oldStatus = original.get("status")
    const newStatus = e.record.get("status")
    
    e.next()
    
    if (oldStatus !== newStatus && newStatus === "approved") {
        // Send notification after successful update
        const user = $app.dao().findRecordById("users", e.record.get("author"))
        
        const message = new MailerMessage({
            from: {
                address: $app.settings().meta.senderAddress,
                name: $app.settings().meta.senderName,
            },
            to: [{address: user.email()}],
            subject: "Your article was approved",
            html: `<p>Great news! Your article has been approved.</p>`
        })
        
        $app.newMailClient().send(message)
    }
}, "articles")
```

## Cascading Updates

### Update Related Record Count
```javascript
onRecordAfterCreateSuccess((e) => {
    // Increment comment count on article
    const articleId = e.record.get("article")
    const article = $app.dao().findRecordById("articles", articleId)
    
    article.set("commentCount", article.get("commentCount") + 1)
    $app.dao().withoutHooks().saveRecord(article)
    
    e.next()
}, "comments")

onRecordAfterDeleteSuccess((e) => {
    // Decrement comment count on article
    const articleId = e.record.get("article")
    const article = $app.dao().findRecordById("articles", articleId)
    
    article.set("commentCount", Math.max(0, article.get("commentCount") - 1))
    $app.dao().withoutHooks().saveRecord(article)
    
    e.next()
}, "comments")
```

### Propagate Status Changes
```javascript
onRecordAfterUpdateSuccess((e) => {
    const original = e.record.originalCopy()
    
    // If project status changed to "archived", archive all tasks
    if (e.record.get("status") === "archived" && original.get("status") !== "archived") {
        const tasks = $app.dao().findRecordsByFilter(
            "tasks",
            `project = {:projectId}`,
            "",
            0,
            {"projectId": e.record.id}
        )
        
        tasks.forEach(task => {
            task.set("status", "archived")
            $app.dao().withoutHooks().saveRecord(task)
        })
    }
    
    e.next()
}, "projects")
```

### Update Parent Timestamp
```javascript
onRecordAfterCreateSuccess((e) => {
    // Update parent's "lastActivity" when child is created
    const parentId = e.record.get("parent")
    if (parentId) {
        const parent = $app.dao().findRecordById("threads", parentId)
        parent.set("lastActivity", new Date().toISOString())
        $app.dao().withoutHooks().saveRecord(parent)
    }
    
    e.next()
}, "comments")
```

## Soft Deletes

### Implement Soft Delete
```javascript
onRecordDeleteRequest((e) => {
    // Instead of deleting, mark as deleted
    e.record.set("deleted", true)
    e.record.set("deletedAt", new Date().toISOString())
    $app.dao().saveRecord(e.record)
    
    // Don't call e.next() to prevent actual deletion
}, "important_records")
```

### Filter Soft-Deleted from Lists
```javascript
onRecordsListRequest((e) => {
    // Filter out soft-deleted records
    // (Better to do this with API rules: deleted = false || @request.auth.role = "admin")
    e.records = e.records.filter(record => !record.get("deleted"))
    
    e.next()
}, "important_records")
```

## Audit Logging

### Log All Changes
```javascript
onRecordUpdate((e) => {
    const original = JSON.parse(JSON.stringify(e.record.originalCopy()))
    const current = JSON.parse(JSON.stringify(e.record))
    
    const changes = {}
    for (const key in original) {
        if (original[key] !== current[key]) {
            changes[key] = {
                old: original[key],
                new: current[key]
            }
        }
    }
    
    e.next()
    
    // Log after successful update
    if (Object.keys(changes).length > 0) {
        const auditLog = new Record($app.dao().findCollectionByNameOrId("audit_logs"))
        auditLog.set("recordId", e.record.id)
        auditLog.set("collection", "users")
        auditLog.set("action", "update")
        auditLog.set("changes", JSON.stringify(changes))
        auditLog.set("timestamp", new Date().toISOString())
        
        $app.dao().saveRecord(auditLog)
    }
}, "users")
```

### Track Who Changed What
```javascript
onRecordUpdateRequest((e) => {
    const authUser = e.httpContext.get("authRecord")
    
    e.record.set("lastModifiedBy", authUser.id)
    e.record.set("lastModifiedAt", new Date().toISOString())
    
    e.next()
}, "documents")
```

## File Management

### Auto-Delete Old File on Update
```javascript
onRecordUpdate((e) => {
    const original = e.record.originalCopy()
    const oldAvatar = original.get("avatar")
    const newAvatar = e.record.get("avatar")
    
    e.next()
    
    // Delete old avatar file if it changed
    if (oldAvatar && oldAvatar !== newAvatar) {
        try {
            $app.dao().deleteFileById(original, oldAvatar)
        } catch (err) {
            console.error("Failed to delete old avatar:", err)
        }
    }
}, "users")
```

### Validate File Types
```javascript
onRecordCreateRequest((e) => {
    const files = e.httpContext.request().multipartForm()?.file["document"]
    
    if (files && files.length > 0) {
        const file = files[0]
        const allowedTypes = ["application/pdf", "image/jpeg", "image/png"]
        
        if (!allowedTypes.includes(file.header.get("Content-Type"))) {
            throw new Error("Invalid file type. Only PDF, JPEG, and PNG allowed.")
        }
    }
    
    e.next()
}, "documents")
```

### Clean Up Files on Delete
```javascript
onRecordAfterDeleteSuccess((e) => {
    // Clean up associated files
    const files = e.record.get("attachments") || []
    
    files.forEach(filename => {
        try {
            $app.dao().deleteFileById(e.record, filename)
        } catch (err) {
            console.error(`Failed to delete file ${filename}:`, err)
        }
    })
    
    e.next()
}, "posts")
```

## Advanced Patterns

### Conditional Hook Logic
```javascript
onRecordCreate((e) => {
    const recordType = e.record.get("type")
    
    // Different logic based on type
    if (recordType === "premium") {
        e.record.set("features", ["feature1", "feature2", "feature3"])
    } else if (recordType === "basic") {
        e.record.set("features", ["feature1"])
    }
    
    e.next()
}, "subscriptions")
```

### Transaction-Safe Updates
```javascript
onRecordAfterCreateSuccess((e) => {
    // This runs after transaction is committed, safe for external calls
    
    try {
        // Call external API
        const response = $http.send({
            url: "https://api.example.com/notify",
            method: "POST",
            body: JSON.stringify({
                recordId: e.record.id,
                type: e.record.get("type")
            }),
            headers: {"Content-Type": "application/json"}
        })
        
        // Log response
        console.log("External API response:", response.statusCode)
    } catch (err) {
        console.error("External API call failed:", err)
        // Don't throw - record was already created successfully
    }
    
    e.next()
}, "events")
```

### Batch Operations
```javascript
onRecordCreate((e) => {
    // When a team is created, create default roles
    const teamId = e.record.id
    
    e.next()
    
    const defaultRoles = ["admin", "member", "viewer"]
    
    defaultRoles.forEach(roleName => {
        const role = new Record($app.dao().findCollectionByNameOrId("roles"))
        role.set("name", roleName)
        role.set("team", teamId)
        $app.dao().saveRecord(role)
    })
}, "teams")
```

## Tips

1. **Use *AfterSuccess hooks for external operations** - They run after transaction commit
2. **Use withoutHooks() to prevent infinite loops** - When updating records in hooks
3. **Parse records to JSON for easy comparison** - `JSON.parse(JSON.stringify(record))`
4. **Use try-catch for external operations** - Don't let external failures break your hooks
5. **Keep hooks focused** - One hook should do one thing well
6. **Test cascading behavior** - Make sure updates don't create unexpected chains
