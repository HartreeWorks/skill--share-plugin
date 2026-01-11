---
name: share-plugin
description: This skill should be used when the user asks to "share a plugin", "make a plugin public", "publish a plugin", "add plugin to marketplace", or mentions making a Claude Code plugin available publicly. Publishes a plugin to the HartreeWorks plugins marketplace.
---

# Share Plugin

This skill publishes a Claude Code plugin to the HartreeWorks plugins marketplace.

## What it does

1. Validates the plugin exists in the marketplace repo
2. **CRITICAL: Security & privacy review** - checks for credentials and private information
3. Creates a README.md if missing
4. Updates `marketplace.json` to list the plugin
5. Updates the index README
6. Commits and pushes to GitHub

## Directory structure

All plugins live directly in the marketplace repo:

```
~/Documents/www/Claude Plugins/plugins/     # Clone of HartreeWorks/claude-plugins
├── .claude-plugin/
│   └── marketplace.json
├── README.md
├── plugin--project-management/             # Plugins developed here
│   ├── .claude-plugin/plugin.json
│   ├── README.md
│   ├── commands/
│   ├── skills/
│   └── scripts/
└── plugin--another-plugin/
```

No separate repos per plugin - everything is in one marketplace repo.

## Prerequisites

- GitHub CLI (`gh`) must be installed and authenticated
- The plugins repo must be cloned at `~/Documents/www/Claude Plugins/plugins/`

## Workflow

When the user asks to share a plugin, follow these steps:

### Step 1: Gather plugin info

Ask the user:
- Plugin name (folder should be `plugin--{name}` in the plugins directory)
- Category for marketplace (productivity, development, utilities, etc.)

### Step 2: Validate the plugin

```bash
PLUGINS_DIR="$HOME/Documents/www/Claude Plugins/plugins"
PLUGIN_NAME="{name}"
PLUGIN_PATH="$PLUGINS_DIR/plugin--$PLUGIN_NAME"

# Check plugin folder exists
ls "$PLUGIN_PATH"

# Check plugin.json exists
cat "$PLUGIN_PATH/.claude-plugin/plugin.json"
```

If the plugin doesn't exist or has no plugin.json, inform the user and stop.

### Step 3: Check if already in marketplace.json

```bash
cat "$PLUGINS_DIR/.claude-plugin/marketplace.json" | grep -q "\"$PLUGIN_NAME\"" && echo "Already listed" || echo "Not yet listed"
```

If already listed, inform the user and stop.

### Step 4: Security & privacy review (CRITICAL)

**This step is mandatory. Do NOT proceed to publishing without completing this review.**

#### 4a: Check for sensitive files

Look for files that might contain credentials or secrets:

```bash
find "$PLUGIN_PATH" -type f \( -name "*.env" -o -name ".env*" -o -name "*secret*" -o -name "*credential*" -o -name "*token*" -o -name "*.key" -o -name "*.pem" \)
```

**Files that MUST NOT be committed:**
- `.env` or any `.env.*` files
- Any file with "secret", "credential", or "token" in the name
- Private keys (`.key`, `.pem`)
- Cache files with user data

**Ensure `.gitignore` excludes:**
- `.DS_Store`
- `.claude/`
- `__pycache__/`
- `node_modules/`
- `*.pyc`

#### 4b: Scan for private information

Read through ALL text files, especially:
- SKILL.md files
- Command .md files
- Script files with examples
- README.md

**Look for:**

| Type | Examples | Replacement |
|------|----------|-------------|
| Client company names | Acme Corp, etc. | Generic names |
| Real people's names | John Smith | Alice, Bob |
| Email addresses | john@client.com | alice@example.com |
| Slack workspace names | client-workspace | example-workspace |
| Phone numbers | Real numbers | +44 20 1234 5678 |
| URLs with client domains | client.slack.com | example.slack.com |

#### 4c: Report findings and get approval

Present findings clearly and use AskUserQuestion:
```
question: "I've completed the security review. Should I make the suggested changes and proceed?"
header: "Review"
options:
  - label: "Apply changes & proceed"
    description: "Make all suggested replacements and continue publishing"
  - label: "Show me the changes first"
    description: "Display the exact edits before applying"
  - label: "Stop - I'll review manually"
    description: "Abort so you can review and edit files yourself"
```

**Only proceed after user explicitly approves.**

### Step 5: Create README.md (if missing)

If no README.md exists in the plugin folder, create one:

```markdown
# {Plugin Name}

{Description from plugin.json}

## Installation

1. Add the HartreeWorks plugins marketplace:
```
/plugin marketplace add hartreeworks/claude-plugins
```

2. Install this plugin:
```
/plugin install {plugin-name}@hartreeworks-plugins
```

## Features

{List commands and skills from the plugin}

## About

Created by [Peter Hartree](https://x.com/peterhartree).

Find more plugins at [HartreeWorks/claude-plugins](https://github.com/HartreeWorks/claude-plugins).
```

### Step 6: Update marketplace.json

Read the current marketplace.json:

```bash
cat "$PLUGINS_DIR/.claude-plugin/marketplace.json"
```

Add a new entry to the `plugins` array:

```json
{
  "name": "{plugin-name}",
  "description": "{description from plugin.json}",
  "source": "./plugin--{plugin-name}",
  "category": "{category}"
}
```

Keep the array sorted alphabetically by name.

### Step 7: Update index README

Edit `$PLUGINS_DIR/README.md` to add a row to the "Available plugins" table:

```markdown
| plugin-name | {brief description} |
```

Keep the table sorted alphabetically.

### Step 8: Commit and push

```bash
cd "$PLUGINS_DIR"
git add .
git commit -m "Add {plugin-name} plugin

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
git push origin main
```

### Step 9: Confirm success

Output:

```
✓ Plugin "{plugin-name}" is now public!

Marketplace: https://github.com/HartreeWorks/claude-plugins

Install with:
  /plugin marketplace add hartreeworks/claude-plugins
  /plugin install {plugin-name}@hartreeworks-plugins
```

## Error handling

| Error | Cause | Solution |
|-------|-------|----------|
| Plugin folder not found | Wrong path | Create plugin in `plugins/plugin--{name}/` first |
| No plugin.json | Not a valid plugin | Create `.claude-plugin/plugin.json` first |
| Already in marketplace | Already shared | Plugin is already listed |
| Privacy review failed | User chose to stop | User reviews manually |
| gh auth error | Not logged in | Run `gh auth login` |

## Example usage

User: "Share the project-management plugin"

1. Validate: `plugins/plugin--project-management` exists with plugin.json ✓
2. Check not already in marketplace.json ✓
3. Security review:
   - Scan for sensitive files
   - Scan text for private info
   - Report and get approval
4. User approves
5. Ensure README.md exists
6. Update marketplace.json
7. Update README.md in index
8. Commit and push
9. Report success

## Notes

- All plugins live in `~/Documents/www/Claude Plugins/plugins/` (clone of HartreeWorks/claude-plugins)
- Plugin folder naming: `plugin--{plugin-name}`
- **Always complete security review before publishing**

## Update check

This is a shared skill. Before executing, check `~/.claude/skills/.update-config.json`.
If `auto_check_enabled` is true and `last_checked_timestamp` is older than `check_frequency_days`,
mention: "It's been a while since skill updates were checked. Run `/update-skills` to see available updates."
Do NOT perform network operations - just check the local timestamp.
