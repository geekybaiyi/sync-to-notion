# Feature 1 Phase 1: Implementation Plan - No-Code Calendar Sync

## No-Code Implementation Philosophy

**CONSTRAINT**: This feature is implemented using **ONLY**:
- ✅ **Natural Language Prompts** - All logic expressed conversationally
- ✅ **MCP Tool Calls** - Notion API access via MCP gateway
- ✅ **AI Reasoning** - Built-in parsing, matching, and logic
- ❌ **NO custom scripts, code files, or external dependencies**

Every operation must be achievable through prompts that trigger appropriate MCP tool calls in sequence.

## Updated Scope: Full Task Integration

**What Phase 1 DOES:**
- ✅ CSV calendar events → Notion Tasks
- ✅ Link Tasks to existing Projects (when found)
- ✅ **Link Tasks to Month/Week/Daily Log entries** (Added)
- ✅ Smart caching optimized by database update frequency
- ✅ Comprehensive linking status reporting

**What Phase 1 DOES NOT:**
- ❌ Create Timesheet entries
- ❌ Create new Projects (report only)
- ❌ Link to Goals (handled by Notion automation)

## Updated CSV Format

### Column Order:
```csv
title,date,start_time,end_time,description,project
"Sprint Planning","2025-01-17","10:00","11:00","Sprint 3 kickoff","Development"
"Doctor Checkup","2025-01-18","14:00","15:30","Annual physical","Health"
"Team Call","2025-01-19","10:00","11:00","Weekly sync call",""
```

## Smart Caching Strategy by Update Frequency

### Cache Tier 1: Daily Cache (Projects)
```
Database: Projects (ID: 15b1eee3-fc62-47ce-8a85-5e41ca832d07)
Update Frequency: Weekly/Monthly (low frequency)
Cache Strategy: Load once per day, persist across sessions
Cache Key: "projects_cache_2025_01_17"
Reload Triggers: 
- Date change
- Manual refresh command
- Cache age > 24 hours
```

### Cache Tier 2: Session Cache (Time Periods)
```
Database: Current Month (ID: 4ee6df88-6558-48b4-862d-2aaa1328ccd0)
Database: Current Week (ID: 8ab06911-3877-4a07-8e75-040c46c560e5)
Update Frequency: Weekly/Monthly (medium frequency)
Cache Strategy: Load once per session
Cache Key: "current_periods_session_123"
Reload Triggers:
- New conversation session
- Manual refresh
```

### Cache Tier 3: On-Demand (Daily Logs)
```
Database: Daily Logs (ID: 1d3552d6-4163-4835-9b4e-08e9cb27e235)
Update Frequency: Daily (high frequency)
Cache Strategy: Query specific dates only, no pre-loading
Rationale: Too frequent updates, better to query as needed
```

## No-Code Workflow Implementation

### Step 1: Cache Initialization (Prompt-Based)

**User Command:**
```
"Initialize calendar sync workspace"
```

**Agent Workflow (Pure Prompts + MCP):**
```
1. Check conversation memory for existing cache
2. If Projects cache missing or > 24 hours old:
   - Use MCP to query Projects database
   - Filter for active projects only
   - Store in conversation context with timestamp
3. Query current Month entry via MCP
4. Query current Week entry via MCP
5. Store time periods in session memory
6. Report cache status to user
```

**MCP Call Sequence:**
```json
[
  {
    "step": "projects_cache",
    "tool": "post-database-query",
    "database_id": "15b1eee3-fc62-47ce-8a85-5e41ca832d07",
    "filter": {
      "and": [
        {"property": "Status", "select": {"does_not_equal": "Completed"}},
        {"property": "Status", "select": {"does_not_equal": "Pause"}}
      ]
    },
    "page_size": 100
  },
  {
    "step": "current_month", 
    "tool": "post-database-query",
    "database_id": "4ee6df88-6558-48b4-862d-2aaa1328ccd0",
    "filter": {
      "property": "isCurrent",
      "formula": {"checkbox": {"equals": true}}
    }
  },
  {
    "step": "current_week",
    "tool": "post-database-query",
    "database_id": "8ab06911-3877-4a07-8e75-040c46c560e5",
    "filter": {
      "property": "isCurrent", 
      "formula": {"checkbox": {"equals": true}}
    }
  }
]
```

### Step 2: CSV Processing (AI Reasoning)

**User Command:**
```
"Sync these calendar events:"
[CSV data]
```

**Agent Workflow (No Custom Code):**
```
1. Parse CSV using built-in text processing
2. Extract events into structured format:
   - title, date, start_time, end_time, description, project
3. Validate required fields (title, date, start_time)
4. Generate Event IDs using SHA256 logic: 
   - hash(date + start_time + title)[0:8]
5. Proceed to event processing
```

### Step 3: Event Processing with Full Linking

**For Each Event (Prompt-Driven Logic):**

**A. Duplicate Check:**
```json
{
  "tool": "post-database-query",
  "database_id": "eec0f1cd-7b83-40fe-a558-17b106406ef4",
  "filter": {
    "property": "Event ID",
    "rich_text": {"equals": "generated_event_id"}
  }
}
```

**B. Project Matching (Using Cache):**
```
AI Reasoning Process:
1. If project column provided:
   - Search cached projects for fuzzy match
   - Use contains/similarity logic
2. If project column empty:
   - Extract keywords from event title
   - Search cached projects for keyword matches
3. If no match found:
   - Log unmatched project
   - Proceed without project link
```

**C. Time Period Linking:**

**Daily Log Lookup:**
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

**Week/Month Linking:**
```
AI Reasoning Process:
1. Extract event date (2025-01-17)
2. Check cached current Week date range
3. If event falls within range: Use cached Week ID
4. If outside range: Query Weeks database
5. Repeat for Month linking
```

**D. Task Creation with Full Relations:**
```json
{
  "tool": "post-page",
  "parent": {"database_id": "eec0f1cd-7b83-40fe-a558-17b106406ef4"},
  "properties": {
    "Name": {"title": [{"text": {"content": "event_title"}}]},
    "Event ID": {"rich_text": [{"text": {"content": "generated_id"}}]},
    "Do Time": {"date": {"start": "2025-01-17T10:00:00"}},
    "Project": {"relation": [{"id": "project_id_if_found"}]},
    "Scheduled Do Day": {"relation": [{"id": "daily_log_id_if_found"}]},
    "Scheduled Week": {"relation": [{"id": "week_id"}]},
    "Scheduled Month": {"relation": [{"id": "month_id"}]},
    "Type": {"multi_select": [{"name": "Discussion/Meeting"}]},
    "Status": {"select": {"name": "In Progress"}},
    "Summary": {"rich_text": [{"text": {"content": "Calendar event: description"}}]}
  }
}
```

## Event Type Classification (AI Reasoning)

**Prompt-Based Classification Logic:**
```
AI analyzes event title and assigns Type based on keywords:

Meeting Detection:
- Keywords: "meeting", "sync", "standup", "call", "discussion"
- Type: "Discussion/Meeting"

Planning Detection:
- Keywords: "planning", "review", "retrospective", "strategy"
- Type: "Planning & Review"

Health Detection:
- Keywords: "doctor", "dentist", "appointment", "checkup", "therapy"
- Type: "Dr. Appt"

Exercise Detection:
- Keywords: "gym", "workout", "exercise", "run", "yoga"
- Type: "Exercise - 有氧"

Social Detection:
- Keywords: "lunch", "dinner", "hangout", "social"
- Type: "Hangout"

Default Fallback:
- Work context: "Discussion/Meeting"  
- Personal context: "Routine - Personal"
```

## Project Matching Algorithm (No Code)

**Cache-Based Fuzzy Matching:**
```
Step 1: CSV Project Column Match
Input: project="Development"
Process: Search cached projects where Name contains "Development"
Logic: Use AI text similarity matching
Result: Found "Software Development Project" → Link

Step 2: Auto-Match (Empty Project Column)
Input: title="Sprint Planning Meeting", project=""
Process: 
- Extract keywords: ["Sprint", "Planning"]
- Search cached projects for keyword matches
- Use AI semantic similarity 
Result: Found "Agile Sprint Management" → Link

Step 3: No Match Protocol
Input: project="InvalidProject"  
Process: Search cache, no results
Logic: Don't create new project (per requirements)
Result: Create task without project link, report unmatched
```

## Comprehensive Response Format

**Complete Sync Report:**
```
✅ Calendar Sync Complete - January 17-19, 2025

🔄 Cache Performance:
• Projects: 23 active projects (cached 2h ago) 
• Current Month: January 2025 (session cache)
• Current Week: Week 3, Jan 13-19 (session cache)
• API calls saved: 18 (vs 27 without cache)

📊 Processing Summary:
• 3 events processed
• 3 new tasks created  
• 0 duplicates skipped

📋 Detailed Results:

🎯 "Sprint Planning" (Jan 17, 10:00 AM)
  ✅ Project: Software Development Project (matched: "Development")
  ✅ Daily Log: January 17, 2025
  ✅ Week: Week 3 (Jan 13-19, 2025)
  ✅ Month: January 2025
  ✅ Type: Discussion/Meeting

🏥 "Doctor Checkup" (Jan 18, 2:00 PM)
  ✅ Project: Health & Wellness (matched: "Health")
  ❌ Daily Log: Not found for Jan 18, 2025
  ✅ Week: Week 3 (Jan 13-19, 2025)  
  ✅ Month: January 2025
  ✅ Type: Dr. Appt

📞 "Team Call" (Jan 19, 10:00 AM)
  ❌ Project: No match found (auto-match failed)
  ✅ Daily Log: January 19, 2025
  ✅ Week: Week 3 (Jan 13-19, 2025)
  ✅ Month: January 2025
  ✅ Type: Discussion/Meeting

⚠️ Issues Found:
• Missing Daily Log for Jan 18, 2025 (create manually if needed)
• No project match for "Team Call" (consider adding project hint)

💡 Performance Notes:
• Projects cache prevented 6 API calls
• Time period cache prevented 12 API calls
• Total execution time: 3.2 seconds

🎯 Next Steps:
• Review unlinked tasks in Notion
• Create missing Daily Log entries as needed
• Update project hints in future CSVs for better matching
```

## Error Handling (No-Code Approach)

### Cache Failures:
```
Prompt Logic:
"If Projects cache load fails:
1. Retry once with exponential backoff
2. If still fails: Proceed without cache (slower)
3. Query projects individually per event
4. Report performance impact to user"
```

### Missing Time Periods:
```
Prompt Logic:
"For each time period link attempt:
1. Query target database for matching date/range
2. If found: Add relation to task
3. If not found: Log missing entry
4. Continue processing (don't fail entire sync)
5. Report missing entries in final summary"
```

### Invalid CSV Format:
```
Prompt Logic:
"Before processing events:
1. Validate CSV has required columns: title,date,start_time,end_time,description,project
2. Check each row has title, date, start_time (minimum)
3. If validation fails: 
   - Show expected format example
   - Request corrected CSV
   - Don't proceed with sync"
```

## Performance Optimization

### API Call Reduction:
```
Without Cache: ~8 calls per event
- 1 duplicate check
- 1 project search  
- 1 daily log search
- 1 week search
- 1 month search
- 1 task creation
- 2 additional queries

With Smart Cache: ~3 calls per event  
- 1 duplicate check
- 1 daily log search (only uncached item)
- 1 task creation

Cache Efficiency: 62% reduction in API calls
```

### Memory Management:
```
Cache Size Estimates:
- Projects (100 items): ~50KB
- Current Month (1 item): ~5KB  
- Current Week (1 item): ~5KB
Total Cache: ~60KB per session

Refresh Strategy:
- Projects: Check age on each session start
- Time Periods: Refresh on session start
- Daily Logs: No caching (query as needed)
```

## Testing Strategy (No-Code Validation)

### Functional Tests:
1. **Full Integration**: Event links to all possible relations
2. **Partial Linking**: Some relations missing (e.g., no Daily Log)
3. **Cache Performance**: Verify Projects cache reduces API calls
4. **Time Period Logic**: Events from different weeks/months link correctly
5. **Project Matching**: Both explicit and auto-matching scenarios
6. **Error Recovery**: Graceful handling of missing data

### No-Code Verification:
- ✅ All logic expressed through prompts
- ✅ No custom code files created
- ✅ Pure MCP tool call sequences
- ✅ AI reasoning handles all parsing/matching
- ✅ Everything reproducible through conversation

### Performance Benchmarks:
- Projects cache loads in < 2 seconds
- 10 events process in < 8 seconds total
- Cache reduces API calls by 60%+
- Memory usage stays under 100MB