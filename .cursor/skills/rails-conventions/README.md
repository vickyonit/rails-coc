# Rails Conventions Skill

This skill helps AI agents (like Cursor, Codex, etc.) follow Rails conventions and best practices when writing Ruby on Rails code.

## Installation

**📖 See [INSTALLATION.md](INSTALLATION.md) for detailed installation instructions.**

### Quick Install Options

**Option 1: Project-Level (Recommended for Teams)**
```bash
cd /path/to/your/rails-project
mkdir -p .cursor/skills
cp -r /path/to/rails-coc/.cursor/skills/rails-conventions .cursor/skills/
# Commit to your repository so team members have access
git add .cursor/skills/
git commit -m "Add Rails Conventions skill"
```

**Option 2: Global (Available Across All Projects)**
```bash
git clone https://github.com/vickyonit/rails-coc.git ~/rails-coc
cp -r ~/rails-coc/.cursor/skills/rails-conventions ~/.cursor/skills/rails-conventions
```

**Option 3: Direct from GitHub**
```bash
cd /path/to/your/rails-project
mkdir -p .cursor/skills
git clone --depth 1 --filter=blob:none --sparse https://github.com/vickyonit/rails-coc.git temp-clone
cd temp-clone && git sparse-checkout set .cursor/skills/rails-conventions
cd .. && mv temp-clone/.cursor/skills/rails-conventions .cursor/skills/
rm -rf temp-clone
```

For more installation options and troubleshooting, see [INSTALLATION.md](INSTALLATION.md).

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
