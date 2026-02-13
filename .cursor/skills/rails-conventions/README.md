# Rails Conventions Skill

This skill helps AI agents (like Cursor, Codex, etc.) follow Rails conventions and best practices when writing Ruby on Rails code.

## Installation

### For Cursor Users

1. Copy this skill directory to your Cursor skills folder:
   ```bash
   cp -r .cursor/skills/rails-conventions ~/.cursor/skills/rails-conventions
   ```

2. The skill will be automatically available in Cursor.

### For Other Agents

If you're using a different AI coding assistant that supports skills:

1. Copy the `.cursor/skills/rails-conventions` directory to your agent's skills location
2. Follow your agent's documentation for skill installation

## What This Skill Does

When enabled, this skill ensures that AI agents:

- ✅ Follow Rails naming conventions (PascalCase classes, snake_case methods, etc.)
- ✅ Use proper model file structure (associations, validations, callbacks in correct order)
- ✅ Write RESTful controllers with proper actions
- ✅ Create migrations following Rails conventions
- ✅ Use appropriate routing patterns
- ✅ Follow Rails best practices and golden rules

## Usage

The skill automatically activates when:
- Working on Rails projects
- Creating models, controllers, views, migrations, or routes
- Writing any Ruby on Rails code

The agent will reference Rails conventions and best practices from the comprehensive Rails CoC documentation files.

## Related Documentation

This skill references the complete Rails Code of Conduct documentation in the parent directory. For detailed information on any specific topic, see:

- [README.md](../../README.md) - Overview and table of contents
- Individual convention files (01-20) - Detailed guides for each topic
