# Feature 1 Phase 1: Calendar to Tasks Sync - Requirements v2

## No Code Implementation Constraint

**CRITICAL**: This feature must be implemented using **ONLY**:
- ✅ **Prompts and conversations** (no custom code files)
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
title,date,start_time,end_time,description,project
"Sprint Planning","2025-01-17","10:00","11:00","Sprint 3 kickoff","Development"
"Doctor Checkup","2025-01-18","14:00","15:30","Annual physical","Health"
"Client Call","2025-01-19","10:00","11:00","Weekly sync call",""
```

### Column Definitions:
- **title**: Event name → Task Name
- **date**: Event date → Task Do Time + Period linking
- **start_time**: Event start → Used for Do Time
- **end_time**: Event end → (Stored but not used in Phase 1)
- **description**: Event details → Task Summary
- **project**: Project hint for matching → Used to find existing project

## Database Linking Strategy

### Primary Database: Tasks
**Tasks Database** (ID: `eec0f1cd-7b83-40fe-a558-17b106406ef4`)
- ✅ **Event ID**: `rich_text` - Duplicate prevention
- ✅ **Name**: `title` - Event title
- ✅ **Do Time**: `date` - Event date/time
- ✅ **Project**: `relation` - Link to Projects DB
- ✅ **Scheduled Do Day**: `relation` - Link to Daily Logs DB
- ✅ **Scheduled Week**: `relation` - Link to Weeks DB  
- ✅ **Scheduled Month**: `relation` - Link to Months DB
- ✅ **Type**: `multi_select` - Event type classification
- ✅ **Status**: `select` - Default: "In Progress"
- ✅ **Summary**: `rich_text` - Event description

### Linking Target Databases
**Daily Logs Database** (ID: `1d3552d6-4163-4835-9b4e-08e9cb27e235`)
- Find by matching event date to Daily Log Date property

**Weeks Database** (ID: `8ab06911-3877-4a07-8e75-040c46c560e5`)
- Find by checking if event date falls within Week Date Range

**Months Database** (ID: `4ee6df88-6558-48b4-862d-2aaa1328ccd0`)
- Find by checking if event date falls within Month Date Range

**Projects Database** (ID: `15b1eee3-fc62-47ce-8a85-5e41ca832d07`)
- For project matching and linking (read-only)

## Smart Caching Strategy

### Cache Levels by Update Frequency

**Level 1: Static Cache (Load once per day)**
```
Projects Database:
- Update frequency: Weekly/Monthly
- Cache duration: 24 hours
- Reload trigger: Date change or manual refresh
```

**Level 2: Session Cache (Load once per session)**
```
Current Month:
- Update frequency: Monthly  
- Cache duration: Session lifetime
- Reload trigger: New chat session

Current Week:
- Update frequency: Weekly
- Cache duration: Session lifetime  
- Reload trigger: New chat session
```

**Level 3: Dynamic Cache (Load as needed)**
```
Daily Logs:
- Update frequency: Daily
- Cache strategy: Query specific dates only
- No pre-loading: Too frequent updates

Goals Database:
- EXCLUDED: Handled by Notion automation
```

### Cache Implementation via MCP

**Session Initialization Prompt:**
```
Load workspace cache for calendar sync:
1. Check if Projects cache exists and is < 24 hours old
2. If not, query Projects database for active projects
3. Query current Month and Week entries  
4. Store in conversation memory for this session
5. Report cache status: "X projects, current month Y, current week Z loaded"
```

## Time Period Linking Logic

### Daily Log Linking
```
For each event:
1. Extract date from CSV (e.g., "2025-01-17")
2. Query Daily Logs database:
   Filter: Date property equals "2025-01-17"
3. If found: Link Task to Daily Log via "Scheduled Do Day" relation
4. If not found: Report "No Daily Log found for 2025-01-17"
```

### Week Linking
```
For each event:
1. Extract date from CSV (e.g., "2025-01-17") 
2. Check cached current Week date range
3. If event date falls within range: Link Task to current Week
4. If outside range: Query Weeks database for matching date range
5. Link via "Scheduled Week" relation
```

### Month Linking
```
For each event:
1. Extract date from CSV (e.g., "2025-01-17")
2. Check cached current Month date range  
3. If event date falls within range: Link Task to current Month
4. If outside range: Query Months database for matching date range
5. Link via "Scheduled Month" relation
```

## No-Code Implementation Workflow

### Step 1: Cache Initialization (Prompt-Driven)
```
User: "Initialize calendar sync cache"

Agent: Uses MCP tools to:
1. Check Projects cache age
2. Query active Projects if needed
3. Query current Month entry
4. Query current Week entry
5. Store results in conversation context
6. Report cache status
```

### Step 2: CSV Processing (Pure Prompt Logic)
```
User: "Sync these calendar events:"
[CSV data]

Agent: 
1. Parse CSV using built-in reasoning
2. Extract events into structured data
3. Generate Event IDs using hash logic
4. Validate required fields
5. Proceed to sync workflow
```

### Step 3: Event Processing (MCP Tool Chain)
```
For each event:
1. Check duplicates via MCP query
2. Match project using cached data + fuzzy logic
3. Find time period entries via MCP queries
4. Create Task via MCP with all relations
5. Report results
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

## MCP Tool Call Sequence

### Cache Loading (Session Start)
```json
{
  "cache_projects": {
    "tool": "post-database-query",
    "database_id": "15b1eee3-fc62-47ce-8a85-5e41ca832d07",
    "filter": {
      "and": [
        {"property": "Status", "select": {"does_not_equal": "Completed"}},
        {"property": "Status", "select": {"does_not_equal": "Pause"}}
      ]
    }
  },
  "cache_current_month": {
    "tool": "post-database-query",
    "database_id": "4ee6df88-6558-48b4-862d-2aaa1328ccd0",
    "filter": {
      "property": "isCurrent",
      "formula": {"checkbox": {"equals": true}}
    }
  },
  "cache_current_week": {
    "tool": "post-database-query", 
    "database_id": "8ab06911-3877-4a07-8e75-040c46c560e5",
    "filter": {
      "property": "isCurrent",
      "formula": {"checkbox": {"equals": true}}
    }
  }
}
```

### Task Creation with Full Linking
```json
{
  "tool": "post-page",
  "parent": {"database_id": "eec0f1cd-7b83-40fe-a558-17b106406ef4"},
  "properties": {
    "Name": {"title": [{"text": {"content": "Sprint Planning"}}]},
    "Event ID": {"rich_text": [{"text": {"content": "abc123ef"}}]},
    "Do Time": {"date": {"start": "2025-01-17T10:00:00"}},
    "Project": {"relation": [{"id": "project_page_id"}]},
    "Scheduled Do Day": {"relation": [{"id": "daily_log_page_id"}]},
    "Scheduled Week": {"relation": [{"id": "week_page_id"}]},
    "Scheduled Month": {"relation": [{"id": "month_page_id"}]},
    "Type": {"multi_select": [{"name": "Discussion/Meeting"}]},
    "Status": {"select": {"name": "In Progress"}},
    "Summary": {"rich_text": [{"text": {"content": "Calendar event: Sprint 3 kickoff"}}]}
  }
}
```

### Daily Log Lookup (Per Event Date)
```json
{
  "tool": "post-database-query",
  "database_id": "1d3552d6-4163-4835-9b4e-08e9cb27e235",
  "filter": {
    "property": "Date",
    "date": {"equals": "2025-01-17"}
  }
}
```

## Expected User Experience

### Input Command
```
User: "Sync these calendar events:"

title,date,start_time,end_time,description,project
"Sprint Planning","2025-01-17","10:00","11:00","Sprint 3 kickoff","Development"
"Doctor Visit","2025-01-18","14:00","15:30","Annual checkup","Health"
"Team Call","2025-01-19","10:00","11:00","Weekly sync",""
```

### Output Response
```
✅ Calendar Sync Complete - January 17-19, 2025

🔄 Cache Status:
• 23 active projects loaded (cached 2 hours ago)
• Current month: January 2025 loaded
• Current week: Week 3 (Jan 13-19) loaded

📊 Summary:
• 3 events processed
• 3 new tasks created
• 0 duplicates skipped

📋 Tasks Created:

• "Sprint Planning" (Jan 17, 10:00 AM)
  ✅ Project: Software Development Project
  ✅ Daily Log: Jan 17, 2025
  ✅ Week: Week 3 (Jan 13-19)
  ✅ Month: January 2025

• "Doctor Visit" (Jan 18, 2:00 PM)  
  ✅ Project: Health & Wellness
  ✅ Daily Log: Jan 18, 2025
  ✅ Week: Week 3 (Jan 13-19)
  ✅ Month: January 2025

• "Team Call" (Jan 19, 10:00 AM)
  ❌ No project linked (auto-match failed)
  ✅ Daily Log: Jan 19, 2025
  ✅ Week: Week 3 (Jan 13-19)
  ✅ Month: January 2025

⚠️ Linking Issues:
• No Daily Log found for Jan 18, 2025 (create manually if needed)

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
- ✅ Link to Month/Week/Daily Log entries
- ✅ Report linking status clearly
- ✅ Prevent duplicate tasks
- ✅ Smart caching by database update frequency

### No-Code Constraints:
- ✅ Everything done via prompts + MCP calls
- ✅ No custom code files created
- ✅ All logic expressed in natural language
- ✅ Built-in AI reasoning for parsing/matching

### Performance Requirements:
- ✅ Cache Projects once per day maximum
- ✅ Cache current time periods once per session
- ✅ Process 10 events in < 10 seconds
- ✅ Minimize API calls through smart caching

## Error Handling

### Missing Time Period Entries:
```
Event date: 2025-01-17
Daily Log query: No results
→ Create task without Daily Log link
→ Report: "No Daily Log found for Jan 17, 2025"
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
1. **Full Linking Success**: Event links to Project + Month + Week + Daily Log
2. **Partial Linking**: Event links to some but not all time periods
3. **No Project Match**: Event creates task without project link
4. **Missing Daily Log**: Event date has no corresponding Daily Log entry
5. **Cache Performance**: Verify Projects cache reduces API calls
6. **No-Code Verification**: Ensure no custom code files created

### Performance Validation:
- Projects cache reduces API calls by 80%+
- Current time period cache eliminates redundant queries
- Total sync time scales linearly with event count