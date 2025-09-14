# sync-to-notion

A pure no-code Notion automation system powered by Claude Code sub agents. Intelligently syncs calendar events to Notion databases with AI-powered project matching and smart cache management.

## Features

- **Calendar Event Sync**: CSV to Notion Tasks with intelligent duplicate detection
- **Smart Project Matching**: AI reasoning to match events with appropriate projects
- **Cache Management**: Automated refresh of Projects and Daily Logs data
- **Schema Discovery**: Automatic mapping of Notion database structures
- **No-Code Architecture**: Pure agent-based automation without traditional code dependencies

## Quick Start

### 1. Setup
Ensure you have Claude Code configured with Notion API access.

### 2. Initialize System
Simply ask Claude Code in natural language:
```
"Discover and document my Notion database schemas"
"Refresh my Projects cache"
"Refresh my Daily Logs cache"
```

### 3. Sync Calendar Events
Just tell Claude Code what you want to sync:
```
"Sync this CSV data to my Notion tasks: [paste your CSV data here]"
"Sync the calendar events from events.csv to Notion"
```

## CSV Format

Required fields:
- `title` - Event name
- `date` - Event date (YYYY-MM-DD)
- `start_time` - Start time (HH:MM)
- `end_time` - End time (HH:MM)
- `event_id` - Unique identifier
- `project` - Optional explicit project assignment

## Architecture

- **.claude/agents/**: Claude sub agent specifications
- **output/**: Cache files and documentation
- **CLAUDE.md**: Complete development guide

The system maintains local caches for performance and uses intelligent agents for complex decision-making, replacing traditional code with AI reasoning.

