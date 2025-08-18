# Feature 1 Phase 1: Calendar to Tasks Sync - Updated Requirements

## Scope Changes

### What's INCLUDED in Phase 1:
- ✅ Create Tasks from calendar events
- ✅ Link Tasks to existing Projects (if found)
- ✅ Set proper Task properties (Date, Type, Status)
- ✅ Report sync results with project matching status

### What's EXCLUDED from Phase 1:
- ❌ ~~Create/Update Timesheet entries~~ (Removed)
- ❌ ~~Create new Projects~~ (Report only)
- ❌ ~~Link to Daily Logs~~ (Simplified scope)

## Updated User Input Format

### Preferred CSV Format with Project Column:
```csv
title,date,start_time,end_time,calendar_source,project,description
"Project Alpha Kickoff","2025-01-17","10:00","11:00","Work","Alpha","Initial project meeting"
"Dentist Appointment","2025-01-17","14:00","15:30","Personal","Health","Routine checkup"
"Team Review","2025-01-17","16:00","17:00","Work","Marketing","Weekly sync"
```

### Project Column Usage:
- **Purpose**: Help identify which Notion project to link to
- **Matching**: Fuzzy matching against existing project names
- **Behavior**: If no match found, report but don't create new project
- **Optional**: Can be empty, agent will attempt auto-matching

## Updated Database Requirements

### Primary Database: Tasks Only
**Tasks Database** (ID: `eec0f1cd-7b83-40fe-a558-17b106406ef4`)
- ✅ **Event ID**: `rich_text` - Duplicate prevention
- ✅ **Name**: `title` - Event title
- ✅ **Do Time**: `date` - Event date/time
- ✅ **Project**: `relation` - Link to Projects DB
- ✅ **Type**: `multi_select` - Event type classification
- ✅ **Status**: `select` - Default: "In Progress"
- ✅ **Summary**: `rich_text` - Event description

### Reference Databases (Read-Only for Caching)
**Projects Database** (ID: `15b1eee3-fc62-47ce-8a85-5e41ca832d07`)
- For project matching and linking

**Goals Database** (ID: `bf0e1fef-e975-4cf8-89d6-0cd9bed9aa32`)
- For context and potential future linking

**Months/Weeks/Daily Logs** - For future phase integration

## Project Matching Strategy (Updated)

### 1. CSV Project Column Matching
```
Input project: "Alpha" 
→ Search Projects where Name contains "Alpha"
→ Found: "Project Alpha Development" ✅
```

### 2. Auto-Matching (when project column empty)
```
Event: "Sprint Planning Meeting"
→ Extract keywords: ["Sprint", "Planning"]  
→ Search Projects for keyword matches
→ Found: "Sprint Development Project" ✅
```

### 3. No Match Found
```
Input project: "Unknown Project"
→ Search Projects for "Unknown Project"
→ No matches found ❌
→ Report: "Project not found: Unknown Project"
→ Create task WITHOUT project link
```

## Performance Optimization: Database Caching

### Initial Cache Loading
```json
{
  "strategy": "fetch_on_startup",
  "databases": [
    {
      "name": "Projects",
      "id": "15b1eee3-fc62-47ce-8a85-5e41ca832d07",
      "properties": ["Name", "Status"],
      "filter": {"Status": {"select": {"does_not_equal": "Completed"}}}
    },
    {
      "name": "Goals", 
      "id": "bf0e1fef-e975-4cf8-89d6-0cd9bed9aa32",
      "properties": ["Name", "Status"],
      "filter": {"Status": {"select": {"does_not_equal": "Achieved"}}}
    },
    {
      "name": "Current Month",
      "id": "4ee6df88-6558-48b4-862d-2aaa1328ccd0", 
      "properties": ["Name", "Date Range"],
      "filter": {"isCurrent": {"formula": {"checkbox": {"equals": true}}}}
    },
    {
      "name": "Current Week",
      "id": "8ab06911-3877-4a07-8e75-040c46c560e5",
      "properties": ["Name", "Date Range"], 
      "filter": {"isCurrent": {"formula": {"checkbox": {"equals": true}}}}
    }
  ]
}
```

### Cache Usage Benefits
- **Faster Project Matching**: No real-time queries during sync
- **Reduced API Calls**: Bulk fetch vs individual lookups
- **Better User Experience**: Instant feedback on project availability
- **Future-Ready**: Cache available for Goals/Time period linking

## Updated Workflow

### Step 1: Initialize Cache (Once per session)
```
🔄 Loading workspace context...
• Projects: 23 active projects loaded
• Goals: 8 active goals loaded  
• Current Month: January 2025 loaded
• Current Week: Week 3 loaded
✅ Ready for calendar sync
```

### Step 2: Parse CSV Input
```csv
title,date,start_time,end_time,calendar_source,project,description
"Sprint Planning","2025-01-17","10:00","11:00","Work","Development","Sprint 3 planning"
"Doctor Visit","2025-01-17","14:00","15:30","Personal","Health","Annual checkup"
"Unknown Meeting","2025-01-17","16:00","17:00","Work","NonExistent",""
```

### Step 3: Process Each Event
```
Event 1: "Sprint Planning"
✅ Project match: "Development" → "Software Development Project"
✅ Task created with project link

Event 2: "Doctor Visit"  
✅ Project match: "Health" → "Health & Wellness"
✅ Task created with project link

Event 3: "Unknown Meeting"
❌ Project not found: "NonExistent"
✅ Task created WITHOUT project link
```

### Step 4: Report Results
```
✅ Calendar Sync Complete - January 17, 2025

📊 Summary:
• 3 events processed
• 3 new tasks created
• 0 duplicates skipped

📋 Tasks Created:
• "Sprint Planning" → ✅ Software Development Project
• "Doctor Visit" → ✅ Health & Wellness  
• "Unknown Meeting" → ❌ No project linked

⚠️ Projects Not Found:
• "NonExistent" (for "Unknown Meeting")

💡 Suggestions:
• Check project name spelling
• Verify project exists and is active
• Leave project column empty for auto-matching
```

## Event Type Classification (Simplified)

### Auto-Detection Rules:
```
Keywords → Type Assignment:
• "meeting", "sync", "standup" → "Discussion/Meeting"
• "appointment", "doctor", "dentist" → "Dr. Appt"  
• "gym", "workout", "exercise" → "Exercise - 有氧"
• "planning", "review", "retrospective" → "Planning & Review"
• Default → "Discussion/Meeting"
```

## Error Handling (Updated)

### Project Matching Failures:
```
⚠️ Project not found: "InvalidProject" for event "Team Meeting"
→ Task created without project link
→ Included in final report for user review
```

### Duplicate Events:
```
✅ Event already exists: "Sprint Planning" (Event ID: abc12345)
→ Skipped duplicate entry
→ Counted in summary report
```

### API Errors:
```
❌ Failed to create task: API rate limit exceeded
→ Retry with exponential backoff (1s, 2s, 4s)
→ Report failure if all retries exhausted
```

## Success Criteria (Updated)

### Functional Requirements:
- ✅ Parse CSV with project column
- ✅ Create Tasks with proper properties
- ✅ Link to existing projects when found
- ✅ Report unmatched projects clearly
- ✅ Prevent duplicate tasks
- ✅ Cache databases for performance

### User Experience Requirements:
- ✅ Single CSV input command
- ✅ Clear project matching feedback
- ✅ Actionable error messages
- ✅ Performance under 5 seconds for 10 events

## Phase 2 Considerations

### Future Enhancements (NOT in Phase 1):
- Timesheet entry creation
- Daily Log integration  
- New project creation workflow
- Bulk date range operations
- Recurring event detection

### Architecture Ready for Phase 2:
- Database cache can support additional linking
- Event ID system supports timesheet correlation
- Project matching foundation scalable