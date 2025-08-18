# Feature 1 Phase 1: Calendar to Tasks Sync - Requirements v2

## No Code Implementation Constraint

**CRITICAL**: This feature must be implemented using **ONLY**:
- ✅ **Prompts and conversations** (no custom code files)
- ✅ **Create Gemini Cli custom commands** with appropriate parameters when needed
- ✅ **MCP tool calls** (Notion API via MCP gateway)  
- ✅ **Built-in AI reasoning** (parsing, logic, matching)
- ❌ **NO custom scripts, functions, or code files**
- ❌ **NO external dependencies or installations**

All logic must be expressed through natural language prompts that trigger appropriate MCP tool calls in sequence.

## Updated Scope

### What's INCLUDED in Phase 1:
- ✅ Create Tasks from calendar events
- ✅ Link Tasks to existing Projects (if found)
- ✅ **Link Tasks to current Month/Week/Daily Log** (Added)
- ✅ Set proper Task properties (Date, Type, Status)
- ✅ Report sync results with linking status
- ✅ Smart caching based on database update frequency

### What's EXCLUDED from Phase 1:
- ❌ ~~Create/Update Timesheet entries~~ (Future phase)
- ❌ ~~Create new Projects~~ (Report only)
- ❌ ~~Goal linking~~ (Handled by Notion automation)

## Updated CSV Input Format

### Required Column Order:
```csv
title,date,start_time,end_time,project
"Sprint Planning","2025-01-17","10:00","11:00","Development"
"Doctor Checkup","2025-01-18","14:00","15:30","Health"
"Client Call","2025-01-19","10:00","11:00",""
```

### Column Definitions:
- **title**: Event name → Task Name
- **date**: Event date → Task Do Time + Daily Log linking
- **start_time**: Event start → Used for Do Time
- **end_time**: Event end → (Stored but not used in Phase 1)
- **project**: Project hint for matching → Used to find existing project

## Database Linking Strategy

### Primary Database: Tasks
**Tasks Database** (ID: `eec0f1cd-7b83-40fe-a558-17b106406ef4`)
- ✅ **Event ID**: `rich_text` - Duplicate prevention
- ✅ **Name**: `title` - Event title
- ✅ **Do Time**: `date` - Event date/time
- ✅ **Project**: `relation` - Link to Projects DB
- ✅ **Scheduled Do Day**: `relation` - Link to Daily Logs DB

### Linking Target Databases
**Daily Logs Database** (ID: `1d3552d6-4163-4835-9b4e-08e9cb27e235`)
- Find by matching event date to Daily Log Date property

**Projects Database** (ID: `15b1eee3-fc62-47ce-8a85-5e41ca832d07`)
- For project matching and linking (read-only)

## Caching Strategy

### Cache Levels by Update Frequency

**Level 1: Database-Specific Cache Files (Auto-refresh when stale)**
```
Projects Database:
- Load only active projects (exclude Completed/Pause status)
- Cache file: output/cache_projects.md
- Auto-refresh trigger: Cache file older than 1 week

Daily Logs:
- Load current and next 21 coming daily log pages
- Cache file: output/cache_daily_logs.md  
- Auto-refresh trigger: Cache file older than 1 week

```

**Level 2: Reference Cache (Manual Refresh)**
```
AOF Database:
- Load all AOFs 
- Cache file: output/cache_aof.md

Goals Database:
- Load all active goals
- Cache file: output/cache_goals.md

```

### Cache Implementation via MCP

**Gemini CLI Custom Commands:**
```
Create custom Gemini CLI commands (TOML files) for complete workflow:

Cache Management Commands:
1. "refresh_cache_projects" - Load active projects, save to output/cache_projects.md
2. "refresh_cache_daily_logs" - Load current + next 21 daily logs, save to output/cache_daily_logs.md

Main Sync Command:
3. "sync_calendar_events" - Accept CSV file path or CSV data parameter, perform full calendar sync
   - Support CSV file path input (e.g., --file="events.csv") 
   - Support direct CSV data input (e.g., --csv="title,date...")
   - Auto-check cache age and refresh if stale (1 week)
   - Parse CSV events from file or data
   - Check duplicates, match projects, link Daily Logs
   - Create tasks with full linking
   - Report results

Technical Details:
- Cache entries include all properties except: button, formula, relationship, rollup
- Use Notion MCP tools - AI agent handles MCP calls automatically
- All commands use prompt-based logic, no custom code files
```

## Time Period Linking Logic

### Daily Log Linking (Phase 1 Focus)
```
For each event:
1. Extract date from CSV (e.g., "2025-01-17")
2. Look up the Daily Logs cache to find page with matching Date property
3. If found: Link Task to Daily Log via "Scheduled Do Day" relation
4. If not found: Stop processing and ask user "No Daily Log found for 2025-01-17. Create missing Daily Log entries first?"
5. Event ID generation: Simple concatenation hash(date + start_time + title)
   Example: hash("2025-01-1710:00Sprint Planning")
```


## No-Code Implementation Workflow


### Step 1: Cache Initialization (Prompt-Driven)
```
User: "Refresh calendar sync cache"

Agent: Uses Notion MCP to:
1. Check Projects cache file age (output/cache_projects.md)
2. If stale (>1 week): Query active Projects and save to cache
3. Check Daily Logs cache file age (output/cache_daily_logs.md)
4. If stale (>1 week): Query current + next 21 Daily Logs and save to cache
5. Load cache files into conversation context
6. Report cache status: "X projects, Y daily logs loaded"
```

### Step 2: Calendar Sync (Gemini CLI Command)
```
User: Execute "sync_calendar_events" command with CSV file path or data

Agent (via Gemini CLI): 
1. Auto-check cache age and refresh if stale (1 week)
2. Read CSV from file path OR parse provided CSV data
3. Extract events into structured data
4. Generate Event IDs using hash logic
5. Validate required fields
6. Proceed to sync workflow
```

### Step 3: Event Processing (Notion MCP Integration)
```
For each event:
1. Check duplicates using Notion MCP (query Tasks database for Event ID)
2. Match project using cached data + fuzzy logic (active projects only)
3. Find Daily Log entry in cache by date
4. If Daily Log missing: Stop and ask user to create missing entries
5. Create Task using Notion MCP with Project + Daily Log relations
6. Report results with linking status
```

## Project Matching Strategy (No New Creation)

### 1. CSV Project Column Matching
```
Input project: "Development"
→ Search cached Projects where Name contains "Development"
→ Found: "Software Development Project" ✅
→ Link Task to found project
```

### 2. Auto-Matching (when project column empty)
```
Event: "Sprint Planning Meeting"
→ Extract keywords using AI reasoning: ["Sprint", "Planning"]
→ Search cached Projects for keyword matches
→ Found: "Agile Sprint Project" ✅
→ Link Task to matched project
```

### 3. No Match Found (Report Only)
```
Input project: "Unknown Project"
→ Search cached Projects for "Unknown Project"
→ No matches found ❌
→ Create task WITHOUT project link
→ Report: "Project not found: Unknown Project"
```


## Expected User Experience

### Input Command (Gemini CLI)
```
Option 1: CSV File Path
User: gemini sync_calendar_events --file="events.csv"

Option 2: Direct CSV Data
User: gemini sync_calendar_events --csv="
title,date,start_time,end_time,project
Sprint Planning,2025-01-17,10:00,11:00,Development
Doctor Visit,2025-01-18,14:00,15:30,Health
Team Call,2025-01-19,10:00,11:00,
"

Option 3: Interactive Prompt
User: gemini sync_calendar_events
[Then provide file path or paste CSV when prompted]
```

### Output Response
```
✅ Calendar Sync Complete - January 17-19, 2025

🔄 Cache Status:
• 23 active projects loaded (cached 3 days ago)
• 22 daily logs loaded (current + next 21 days, cached 5 days ago)

📊 Summary:
• 3 events processed
• 3 new tasks created
• 0 duplicates skipped

📋 Tasks Created:

• "Sprint Planning" (Jan 17, 10:00 AM)
  ✅ Project: Software Development Project
  ✅ Daily Log: Jan 17, 2025

• "Doctor Visit" (Jan 18, 2:00 PM)  
  ✅ Project: Health & Wellness
  ✅ Daily Log: Jan 18, 2025

• "Team Call" (Jan 19, 10:00 AM)
  ❌ No project linked (auto-match failed)
  ✅ Daily Log: Jan 19, 2025

⚠️ Linking Issues:
• No Daily Log found for Jan 18, 2025 - Processing stopped

💡 Suggestions:
• Review unlinked tasks in Notion
• Create missing Daily Log entries
• Consider updating project hints in CSV
```

## Success Criteria

### Functional Requirements:
- ✅ Parse CSV with updated column order
- ✅ Create Tasks with full time period linking
- ✅ Link to existing Projects when found
- ✅ Link to Daily Log entries
- ✅ Report linking status clearly
- ✅ Prevent duplicate tasks
- ✅ Smart caching 

### No-Code Constraints:
- ✅ Everything done via prompts + MCP calls
- ✅ No custom code files created
- ✅ All logic expressed in natural language
- ✅ Built-in AI reasoning for parsing/matching


## Error Handling

### Missing Daily Log Entries:
```
Event date: 2025-01-17
Daily Log cache lookup: No matching page found
→ Stop processing remaining events
→ Ask user: "No Daily Log found for Jan 17, 2025. Create missing Daily Log entries first?"
→ User must create missing entries before retrying sync
```

### Project Not Found:
```
Project hint: "InvalidProject"
→ Search cached projects
→ No matches found
→ Create task without project link  
→ Report: "Project not found: InvalidProject"
```

### Cache Failures:
```
Projects cache load fails
→ Retry once
→ If still fails: Proceed without cache
→ Query projects per-event (slower but functional)
→ Report: "Working without cache - performance reduced"
```

## Testing Strategy

### Core Test Cases:
1. **Full Linking Success**: Event links to Project + Daily Log
2. **Project Only**: Event links to Project but not Daily Log (should stop processing)
3. **No Project Match**: Event creates task without project link but with Daily Log
4. **Missing Daily Log**: Event date has no corresponding Daily Log entry (should stop and ask user)
5. **Cache Performance**: Verify cache files reduce Notion API calls
6. **No-Code Verification**: Ensure no custom code files created, only Gemini CLI commands

### Performance Validation:
- Projects cache reduces API calls by 80%+
- Daily Logs cache eliminates redundant queries for date lookups
- Auto-refresh prevents stale cache issues
- Total sync time scales linearly with event count