# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **pure no-code Notion automation system** that syncs calendar events to Notion databases using intelligent sub agents. The system maintains local markdown caches for performance and uses AI reasoning for smart project matching and data processing.

## Core Architecture

### Data Flow
1. **Schema Discovery** → Documents Notion database structures
2. **Cache Refresh** → Pulls filtered data to local markdown files
3. **Calendar Sync** → Processes CSV events against cached data → Creates/updates Notion tasks

### Key Directories
- `.claude/agents/` - Claude sub agent specifications
- `output/` - Cache files and schema documentation
- `.gemini/` - Legacy Gemini CLI commands (reference only)

## Available Sub Agents

### 1. Cache Refresher Agent
**Name**: `cache-refresher`
**Purpose**: Refreshes Notion database caches
**Usage**: "Refresh my Projects cache" or "Refresh my Daily Logs cache"
**Output**: Updates `output/cache_projects.md` or `output/cache_daily_logs.md`

### 2. Schema Discoverer Agent
**Name**: `schema-discoverer`
**Purpose**: Maps Notion database schemas
**Usage**: "Discover and document my Notion database schemas"
**Output**: Updates `output/reference_notion_schema.md`

### 3. Calendar Syncer Agent
**Name**: `calendar-syncer`
**Purpose**: Main sync workflow - CSV events to Notion Tasks
**Usage**: "Sync this CSV data to my Notion tasks: [data]" or "Sync events.csv to Notion"
**Features**: AI project matching, cache management, conflict detection

## Critical Database IDs

- **Tasks**: `eec0f1cd-7b83-40fe-a558-17b106406ef4`
- **Projects**: `15b1eee3-fc62-47ce-8a85-5e41ca832d07`
- **Daily Logs**: `1d3552d6-4163-4835-9b4e-08e9cb27e235`
- **Areas of Focus**: `9edd1d50-1551-458e-b9af-0d6b47d8425f`

## Cache Management

### Cache Files
- `output/cache_projects.md` - Active projects (YR2025, Active/Light-Touch status)
- `output/cache_daily_logs.md` - Daily logs (next 21 days)
- `output/reference_notion_schema.md` - Complete database schemas

### Cache Refresh Logic
- Projects: Filter by Year="YR2025" AND (Status="Active" OR Status="Light-Touch")
- Daily Logs: Date >= today AND Date <= today+21 days
- Auto-refresh: If cache is older than 1 week during sync operations

## CSV Event Format

Required fields for calendar sync:
- `title` - Event name (becomes Task name)
- `date` - Event date
- `start_time` - Event start time
- `end_time` - Event end time
- `event_id` - Unique identifier (prevents duplicates)
- `project` - Optional explicit project assignment

## Common Workflows

### Full System Setup
1. Run schema-discoverer agent to map all databases
2. Run cache-refresher agent with "Projects"
3. Run cache-refresher agent with "DailyLogs"
4. System ready for calendar sync operations

### Daily Calendar Sync
1. Prepare CSV with required fields
2. Run calendar-syncer agent with CSV data
3. Agent auto-refreshes caches if needed
4. Review sync report for any issues

### Troubleshooting Cache Issues
1. Check cache file timestamps in `output/`
2. Manual refresh: Run cache-refresher for specific database
3. Schema issues: Run schema-discoverer to update references

## Error Handling

### Common Issues
- **Missing Daily Logs**: Sync stops, user must create missing entries
- **Project Matching**: AI reasoning suggests matches, manual review may be needed
- **API Failures**: Agents retry with exponential backoff
- **Cache Staleness**: Auto-refresh during sync operations

### Debug Information
- Error logs in `debug.log`
- Agent reports include detailed status information
- Cache files show last refresh timestamps

## Development Notes

### No Traditional Code
- No package.json, requirements.txt, or build processes
- No version dependencies or technical debt
- Pure agent-based reasoning and API interactions

### Agent Invocation
Use Claude Code's Task tool to invoke agents:
```
Task tool with subagent_type="general-purpose"
Reference the appropriate agent specification file
Provide required input parameters
```

### Extending Functionality
- Add new agent specs in `agents/` directory
- Follow existing patterns for cache management and error handling
- Maintain backward compatibility with current data structures