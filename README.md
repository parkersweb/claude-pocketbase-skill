# PocketBase Skill for Claude

A comprehensive skill for Claude that provides expert guidance on PocketBase backend development, with specialized focus on API rules (permissions) and JavaScript event hooks.

## Overview

This skill helps Claude (and Claude Code) correctly implement PocketBase permissions and hooks - two areas where LLMs commonly struggle due to their nuanced behavior and gotchas.

## What's Included

### Core Skill (SKILL.md)
- Core workflow for PocketBase development
- Critical concepts for API rules and permissions
- Hook execution chain and timing
- Common pitfalls and debugging strategies

### Reference Files

1. **permission_patterns.md** - 50+ permission rule examples
   - Authentication rules
   - Ownership rules
   - Field validation rules
   - Role-based rules
   - Relationship rules
   - Complex conditions

2. **hook_guide.md** - Comprehensive hook reference
   - Hook lifecycle and execution order
   - Model hooks vs Request hooks
   - When each hook fires
   - Choosing the right hook for your use case

3. **hook_patterns.md** - Practical implementations
   - Auto-population patterns
   - Validation patterns
   - Notification patterns
   - Cascading updates
   - Soft deletes
   - Audit logging
   - File management

4. **api_reference.md** - Quick API lookup
   - DAO methods
   - Record operations
   - HTTP operations
   - Email operations
   - Filter syntax

## Installation

### For Claude.ai

1. Package the skill:
   ```bash
   python scripts/package_skill.py . ./dist
   ```

2. Upload `pocketbase.skill` in Claude.ai profile settings → Skills

3. Enable for your conversations

### For Claude Code

Simply have this repository available when working with PocketBase projects. Claude Code will be able to reference the skill files as needed.

## Key Features

### Permissions (API Rules)

- **Clear mental models**: Rules act as BOTH authorization AND filters
- **Return code behavior**: Why you get 200/400/404/403 in different scenarios
- **Common patterns**: Owner-only, role-based, relationship-based access
- **Debugging tips**: What to check when rules don't work

### Hooks

- **Hook timing**: When exactly each hook fires in the execution chain
- **Model vs Request**: Which to use and why
- **Common pitfalls solved**:
  - Infinite recursion prevention
  - Hooks not triggering (collection names, file extensions)
  - Changes not persisting (before/after e.next())
  - Cascading updates (when to use withoutHooks())

## Why This Skill Exists

PocketBase's permission system and hooks have subtle behaviors that are hard for LLMs to get right:

1. **Rules are filters**: Failed listRule returns empty array, not error
2. **Superusers bypass rules**: Testing as admin masks permission bugs  
3. **Hook timing matters**: Before vs after e.next() affects what persists
4. **Model vs Request hooks**: Easy to use wrong type and wonder why it doesn't fire
5. **Infinite loops**: Updating records in hooks without withoutHooks()

This skill codifies the knowledge from:
- Official PocketBase documentation
- GitHub discussions and issues
- Common mistakes in real-world usage

## Structure

```
pocketbase/
├── SKILL.md                           # Core guidance (loaded when skill triggers)
└── references/                        # Loaded as needed
    ├── permission_patterns.md         # Permission examples
    ├── hook_guide.md                  # Hook reference
    ├── hook_patterns.md               # Hook implementations
    └── api_reference.md               # API quick reference
```

## Usage Examples

**Setting permissions:**
```
"Create API rules so users can only edit their own posts but anyone can read them"
```

**Implementing hooks:**
```
"Create a hook that sends an email when a post's status changes to 'approved'"
```

**Debugging:**
```
"Why isn't my onRecordUpdate hook firing when I update records via the API?"
```

**Complex scenarios:**
```
"Set up a task management system with proper permissions and hooks for auto-updating counts"
```

## Contributing

This skill was created following Anthropic's skill creation guidelines. To modify:

1. Edit the relevant `.md` files
2. Keep SKILL.md lean (under 500 lines)
3. Move detailed content to references/
4. Test with real PocketBase scenarios
5. Re-package with `package_skill.py`

## License

MIT License - feel free to use and modify for your projects.

## Credits

Built with guidance from:
- [PocketBase Documentation](https://pocketbase.io/docs/)
- PocketBase GitHub Discussions
- Real-world PocketBase development experience

Skill created for use with Claude by Anthropic.
