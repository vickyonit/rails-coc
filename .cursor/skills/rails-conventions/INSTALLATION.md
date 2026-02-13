# Installing Rails Conventions Skill

This guide explains how to add the Rails Conventions skill to your Rails project or make it available globally.

## Option 1: Project-Level Skill (Recommended for Teams)

Add the skill directly to your Rails project so all team members can use it:

### Method A: Copy from GitHub

```bash
# Navigate to your Rails project
cd /path/to/your/rails-project

# Create the skills directory if it doesn't exist
mkdir -p .cursor/skills

# Clone just the skill directory from GitHub
git clone --depth 1 --filter=blob:none --sparse https://github.com/vickyonit/rails-coc.git temp-clone
cd temp-clone
git sparse-checkout set .cursor/skills/rails-conventions
cd ..
mv temp-clone/.cursor/skills/rails-conventions .cursor/skills/
rm -rf temp-clone
```

### Method B: Manual Copy

```bash
# Navigate to your Rails project
cd /path/to/your/rails-project

# Create the skills directory
mkdir -p .cursor/skills

# Copy the skill directory
cp -r /path/to/rails-coc/.cursor/skills/rails-conventions .cursor/skills/
```

### Method C: Git Submodule (For keeping it updated)

```bash
# Navigate to your Rails project
cd /path/to/your/rails-project

# Create the skills directory
mkdir -p .cursor/skills

# Add as submodule
git submodule add https://github.com/vickyonit/rails-coc.git temp-submodule
mv temp-submodule/.cursor/skills/rails-conventions .cursor/skills/
rm -rf temp-submodule
git rm --cached temp-submodule 2>/dev/null || true
```

**Note:** If using project-level skills, commit the `.cursor/skills/` directory to your repository so all team members have access.

## Option 2: Personal/Global Skill (Available Across All Projects)

Install the skill globally so it's available in all your Rails projects:

```bash
# Clone the repository
git clone https://github.com/vickyonit/rails-coc.git ~/rails-coc

# Copy the skill to your global Cursor skills folder
cp -r ~/rails-coc/.cursor/skills/rails-conventions ~/.cursor/skills/rails-conventions

# Optional: Keep it updated
cd ~/rails-coc
git pull
cp -r .cursor/skills/rails-conventions ~/.cursor/skills/rails-conventions
```

Or download directly:

```bash
# Create skills directory if needed
mkdir -p ~/.cursor/skills

# Download and extract just the skill
cd ~/.cursor/skills
curl -L https://github.com/vickyonit/rails-coc/archive/refs/heads/master.tar.gz | tar -xz --strip-components=3 rails-coc-master/.cursor/skills/rails-conventions
mv rails-conventions rails-conventions
```

## Option 3: Using GitHub CLI

If you have GitHub CLI installed:

```bash
# Navigate to your Rails project
cd /path/to/your/rails-project

# Create skills directory
mkdir -p .cursor/skills

# Download the skill files
gh repo clone vickyonit/rails-coc temp-repo
cp -r temp-repo/.cursor/skills/rails-conventions .cursor/skills/
rm -rf temp-repo
```

## Verification

After installation, verify the skill is available:

```bash
# Check if the skill directory exists
ls -la ~/.cursor/skills/rails-conventions/  # For global install
# or
ls -la .cursor/skills/rails-conventions/    # For project-level install

# Verify SKILL.md exists
cat ~/.cursor/skills/rails-conventions/SKILL.md  # For global install
# or
cat .cursor/skills/rails-conventions/SKILL.md    # For project-level install
```

## Updating the Skill

### For Project-Level Installation

```bash
cd /path/to/your/rails-project
cd .cursor/skills/rails-conventions
git pull origin master  # If using git submodule
# OR manually copy updated files from the rails-coc repository
```

### For Global Installation

```bash
cd ~/rails-coc
git pull
cp -r .cursor/skills/rails-conventions ~/.cursor/skills/rails-conventions
```

## Troubleshooting

**Skill not working?**
- Ensure the skill is in the correct location: `~/.cursor/skills/rails-conventions/` (global) or `.cursor/skills/rails-conventions/` (project)
- Verify `SKILL.md` exists in the skill directory
- Restart Cursor after installation
- Check that the skill description matches Rails-related tasks

**Want to customize the skill?**
- Edit `SKILL.md` in the skill directory
- The skill will use your customizations immediately (may need Cursor restart)

## Which Option Should I Choose?

- **Option 1 (Project-Level)**: Best for teams who want consistent Rails conventions across a specific project
- **Option 2 (Global)**: Best for individual developers who work on multiple Rails projects
- **Option 3**: Convenient if you already use GitHub CLI
