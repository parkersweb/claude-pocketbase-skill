---
name: pocketbase
description: Comprehensive guide for PocketBase backend development with focus on API rules/permissions and JavaScript event hooks. Use when working with PocketBase collections, setting up access control, implementing server-side hooks, or debugging permission/hook issues in PocketBase applications.
---

# PocketBase Development Skill

Guide for working with PocketBase as a backend, with specialized focus on the two most error-prone areas: API Rules (permissions) and JavaScript Event Hooks.

## Core Workflow

1. **Define schema** - Create collections with appropriate fields
2. **Set API rules** - Configure permissions (listRule, viewRule, createRule, updateRule, deleteRule)
3. **Implement hooks** - Add server-side logic in `pb_hooks/` directory
4. **Test thoroughly** - Verify permissions and hooks trigger correctly

## API Rules (Permissions)

### Critical Concepts

**Rules act as BOTH authorization AND filters:**
- Empty string `""` = locked (superuser only)
- `null` = public access (no restrictions)
- Non-empty expression = conditional access

**Return codes based on rule type:**
- Unsatisfied `listRule` → 200 with empty array
- Unsatisfied `createRule` → 400
- Unsatisfied `viewRule`, `updateRule`, `deleteRule` → 404
- Locked rule with non-superuser → 403

**Superusers bypass ALL rules** - use test accounts for rule validation

### Common Permission Patterns

See [references/permission_patterns.md](references/permission_patterns.md) for detailed examples.

**Key points:**
- Use `@request.auth.id != ""` to require authentication
- Use `@request.auth.id = id` for owner-only access
- Use `@request.body.fieldName` to validate submitted data
- Use `:isset` modifier to check if field was submitted
- Use `:length` modifier for array field item counts
- Chain conditions with `&&` (AND) and `||` (OR)

### Permission Debugging

When permissions fail unexpectedly:
1. Check if testing as superuser (rules are bypassed!)
2. Verify `@request.auth` is populated (user is authenticated)
3. Test rule expression in isolation
4. Check for typos in field names
5. Remember: `@request.body.*` for submitted data, not `@request.data.*` (renamed in v0.23+)

## JavaScript Event Hooks

### Critical Concepts

**Hook files:**
- Create `*.pb.js` files in `pb_hooks/` directory
- Files auto-reload on save (Unix systems only)
- Use `/// <reference path="../pb_data/types.d.ts" />` for typing

**Hook execution chain:**
- ALL handlers must call `e.next()` to continue chain
- Throwing error OR not calling `e.next()` stops execution
- Hooks share signature: `function(e) {}`

### Hook Categories & Timing

See [references/hook_guide.md](references/hook_guide.md) for comprehensive hook reference.

**Model hooks vs Request hooks:**
- **Model hooks** (`onRecordCreate`, etc.) - No request context, triggered by ANY save/delete (API, hooks, console)
- **Request hooks** (`onRecordCreateRequest`, etc.) - Have request context, only trigger from API calls

**Hook timing for operations:**
```
BEFORE e.next() → before validation & DB operation
AFTER e.next() → after validation & DB operation (may not be persisted yet)
*AfterSuccess → after confirmed DB persistence
*AfterError → after DB operation failure
```

### Common Hook Pitfalls

**Problem: Hook doesn't trigger**
- Using wrong hook type (Request vs Model)
- Collection name filter is case-sensitive and must match exactly
- Hook file not saved as `*.pb.js`
- Syntax error preventing hook registration

**Problem: Infinite recursion**
- Calling `e.dao.saveRecord()` inside update hook without `withoutHooks()`
- Solution: Use `e.dao.withoutHooks().saveRecord(record)` to skip hooks

**Problem: Changes in hook not reflected**
- Modifying record AFTER `e.next()` in Request hooks won't affect response
- Solution: Make changes BEFORE `e.next()` or use Model hooks

**Problem: Cascading hooks don't fire**
- When updating Record A inside Record B's hook, Record A's hooks don't trigger
- Solution: Use `e.dao.saveRecord()` (not `e.dao.withoutHooks()`) if you want cascading

### Hook Patterns

See [references/hook_patterns.md](references/hook_patterns.md) for specific examples.

**Common patterns:**
- Auto-populate fields on create
- Validate complex business logic
- Send notifications on changes
- Update related records
- Prevent certain field modifications

## Accessing Record Data

**In hooks:**
```javascript
// Get current value
const value = e.record.get("fieldName")

// Set value
e.record.set("fieldName", newValue)

// Get original (before changes)
const original = e.record.originalCopy()

// Parse to plain object for comparison
const parsed = JSON.parse(JSON.stringify(e.record))
```

**In API rules:**
```
// Current record field
fieldName = "value"

// Submitted request data
@request.body.fieldName = "value"

// Auth user field  
@request.auth.fieldName = "value"

// Related record via relation field
relationField.otherField = "value"
```

## File Structure

```
project/
├── pb_hooks/
│   ├── main.pb.js          # Primary hooks
│   └── additional.pb.js    # Split by domain if needed
├── pb_data/
│   ├── data.db             # SQLite database
│   └── types.d.ts          # TypeScript definitions (auto-generated)
└── pocketbase              # Executable
```

## Additional Resources

- [references/permission_patterns.md](references/permission_patterns.md) - Common permission rules
- [references/hook_guide.md](references/hook_guide.md) - Complete hook reference
- [references/hook_patterns.md](references/hook_patterns.md) - Hook implementation examples
- [references/api_reference.md](references/api_reference.md) - Core API methods
