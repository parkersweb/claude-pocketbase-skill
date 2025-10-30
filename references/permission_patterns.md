# PocketBase Permission Patterns

Common API rule patterns and examples for securing your collections.

## Table of Contents
1. [Authentication Rules](#authentication-rules)
2. [Ownership Rules](#ownership-rules)
3. [Field Validation Rules](#field-validation-rules)
4. [Role-Based Rules](#role-based-rules)
5. [Relationship Rules](#relationship-rules)
6. [Complex Conditions](#complex-conditions)

## Authentication Rules

### Require Any Authenticated User
```
@request.auth.id != ""
```
- Use for: Restricting access to logged-in users only
- Works for: All rule types (list, view, create, update, delete)

### Public Read, Authenticated Write
```
listRule: null
viewRule: null
createRule: @request.auth.id != ""
updateRule: @request.auth.id != ""
deleteRule: @request.auth.id != ""
```

## Ownership Rules

### Owner-Only Access
```
@request.auth.id = user
```
- Use when: Records have a `user` relation field pointing to auth collection
- User can only access their own records

### Owner Can Edit, Others Can Read
```
listRule: null
viewRule: null  
createRule: @request.auth.id != ""
updateRule: @request.auth.id = user
deleteRule: @request.auth.id = user
```

### Create With Auto-Assigned Owner
```
createRule: @request.auth.id != "" && @request.body.user = @request.auth.id
```
- Ensures the `user` field is set to the authenticated user
- Prevents users from creating records "as" someone else

## Field Validation Rules

### Prevent Field Modification
```
updateRule: @request.auth.id = user && @request.body.status:isset = false
```
- Prevents users from modifying the `status` field
- `:isset` modifier checks if field was submitted in request

### Require Specific Field Values on Create
```
createRule: @request.auth.id != "" && @request.body.status = "pending"
```
- Enforces that new records must have `status = "pending"`

### Validate Field Length
```
createRule: @request.auth.id != "" && @request.body.tags:length <= 5
```
- Limits array fields to maximum 5 items
- Works with multiple select, relation, and file fields

### Allow Appending But Not Removing
```
updateRule: @request.auth.id = user && @request.body.tags- :isset = false
```
- Users can add tags with `tags+` but cannot remove with `tags-`

## Role-Based Rules

### Single Role Check
```
@request.auth.role = "admin"
```

### Multiple Roles (OR condition)
```
@request.auth.role = "admin" || @request.auth.role = "moderator"
```

### Role with Ownership Fallback
```
@request.auth.role = "admin" || @request.auth.id = user
```
- Admins can access everything, users can access their own

### Multi-Select Role Field
```
@request.auth.roles ?= "editor"
```
- Use `?=` operator for multi-select fields
- Checks if "editor" is in the roles array

## Relationship Rules

### Access Through Related Record
```
@request.auth.id = team.owner
```
- User must own the related team
- Follows relation field `team` to check its `owner` field

### Check Related Collection
```
@collection.teamMembers.team = team && @collection.teamMembers.user = @request.auth.id
```
- Complex relationship: user must be a team member
- Queries the `teamMembers` junction collection

### Access Based on Multiple Relations
```
@request.auth.id = course.instructor || (@collection.enrollments.course = id && @collection.enrollments.student = @request.auth.id)
```
- Instructors OR enrolled students can access

## Complex Conditions

### Time-Based Access
```
created >= "2024-01-01 00:00:00.000Z" && created <= "2024-12-31 23:59:59.999Z"
```
- Filter by date ranges (dates compared as strings in RFC3339 format)

### Status-Based Filtering
```
listRule: status = "published" || @request.auth.id = author
```
- Public users see published content, authors see their own drafts

### Context-Specific Rules
```
@request.context != "oauth2"
```
- Prevent certain actions during OAuth2 flow
- Available contexts: `default`, `oauth2`, `otp`, `password`, `realtime`, `protectedFile`

### Conditional Field Requirements
```
createRule: @request.auth.id != "" && (@request.body.type != "premium" || @request.auth.subscription = "active")
```
- If creating "premium" type, user must have active subscription

## Combining Operators

### Available Operators
- `=` equal
- `!=` not equal  
- `>` greater than
- `>=` greater or equal
- `<` less than
- `<=` less or equal
- `~` LIKE/contains (wraps in `%` for wildcard)
- `!~` NOT LIKE/not contains
- `?=` any/at least one equals (for arrays)
- `?!=` any/at least one not equals (for arrays)
- `?~` any/at least one contains (for arrays)
- `?!~` any/at least one not contains (for arrays)

### Grouping with Parentheses
```
(@request.auth.role = "admin" || @request.auth.id = user) && status != "deleted"
```

## Special Modifiers

### :isset - Check Field Submission
```
@request.body.fieldName:isset = true   // Field was submitted
@request.body.fieldName:isset = false  // Field was NOT submitted
```
- Only works with `@request.body.*` fields
- Does NOT work with file uploads (technical limitation)

### :length - Check Array Length  
```
fieldName:length = 0           // Empty array
fieldName:length > 0           // Has items
@request.body.tags:length <= 5 // Max 5 items submitted
```
- Works with multi-select, relation, and file fields
- Does NOT work with uploaded files in `@request.body.*` (technical limitation)

## Common Mistakes

❌ **Wrong:**
```
@request.data.fieldName = "value"
```
✅ **Correct (v0.23+):**
```
@request.body.fieldName = "value"
```

❌ **Wrong:** Testing as superuser
✅ **Correct:** Use regular user accounts to test rules

❌ **Wrong:** Expecting 403 on failed listRule
✅ **Correct:** Failed listRule returns 200 with empty array

❌ **Wrong:**
```
@request.auth = user
```
✅ **Correct:**
```
@request.auth.id = user
```

## Tips

1. **Start permissive, then restrict** - Begin with `null` rules and add restrictions incrementally
2. **Test each rule independently** - Use the Admin UI's "Test rule" feature  
3. **Remember rule = filter** - List rules that filter results don't fail, they just return fewer items
4. **Chain conditions carefully** - Use parentheses for clarity with complex OR/AND logic
5. **Field names are case-sensitive** - Double-check collection and field name spelling
