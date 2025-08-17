# Feature 1 Detailed Plan: On-Demand Calendar Sync (Updated)

This document provides a detailed breakdown of the plan for Feature 1, using MCP tools framework and existing Notion schema.

## Phase 1.1: MCP-Based Manual Calendar Sync

This phase uses Notion MCP tools exclusively, with user providing calendar data through chat interface.

### Database Context (From Schema Analysis)

**Available Database IDs:**
- **Projects Database:** `15b1eee3-fc62-47ce-8a85-5e41ca832d07` (from Journals relation)
- **Days Database:** `1d3552d6-4163-4835-9b4e-08e9cb27e235` (from Journals relation)

**Missing Schemas Needed:**
- Tasks Database (ID and properties needed)
- Timesheet Database (ID and properties needed)

### Step 1: User Input Format (Simplified)

**Recommended Format:**
```
Event: Project Alpha Kickoff
Date: 2025-01-17  
Time: 10:00-11:00
Calendar: Work

Event: Dentist Appointment
Date: 2025-01-17
Time: 14:00-15:30  
Calendar: Personal
```

**Alternative CSV Format:**
```csv
start_time,end_time,title,calendar_source
"2025-01-17 10:00","2025-01-17 11:00","Project Alpha Kickoff","Work"
"2025-01-17 14:00","2025-01-17 15:30","Dentist Appointment","Personal"
```

### Step 2: MCP Tool Chain Process

**User Command:** 
```
"Sync these calendar events to Notion:" 
[followed by event data]
```

**Agent Workflow using MCP Tools:**

1. **Parse Input Data**
   - Extract event details from user input
   - Generate unique Event ID: `hash(date + start_time + title)`

2. **Check for Duplicates**
   ```
   Tool: post-database-query
   Database: Tasks Database  
   Filter: Event ID property equals generated hash
   Action: Skip if found, proceed if not found
   ```

3. **Find Matching Project**
   ```
   Tool: post-database-query
   Database ID: 15b1eee3-fc62-47ce-8a85-5e41ca832d07 (Projects)
   Filter: title contains keywords from event title
   Fallback: Create/use "Unassigned" project
   ```

4. **Create Task Entry**
   ```
   Tool: post-page
   Parent: Tasks Database
   Properties:
   - Title: [Event title]
   - Date: [Event date]
   - Project: [Relation to found/created project]
   - Event ID: [Generated unique hash]
   - Type: "Meeting" | "Appointment" | "Block Time"
   ```

5. **Create Timesheet Entry**
   ```
   Tool: post-page
   Parent: Timesheet Database  
   Properties:
   - Task: [Relation to created task]
   - Start Time: [Event start datetime]
   - End Time: [Event end datetime]
   - Date: [Event date]
   ```

6. **Link to Day (Optional)**
   ```
   Tool: post-database-query
   Database ID: 1d3552d6-4163-4835-9b4e-08e9cb27e235 (Days)
   Filter: Date equals event date
   Action: Link task to day if found
   ```

### Step 3: Error Handling & Fallbacks

**Project Matching Failures:**
- Create "Unassigned Events" project if none exists
- Use calendar source to determine Work/Personal project
- Prompt user for manual project assignment

**API Errors:**
- Retry failed operations up to 3 times
- Report specific errors to user
- Continue processing remaining events

**Invalid Input:**
- Provide format examples 
- Parse what's possible, report what failed
- Ask for corrected input for failed events

### Step 4: Response Format

**Success Response:**
```
✅ Calendar Sync Complete
- 3 events processed
- 2 new tasks created  
- 1 duplicate skipped
- 2 timesheet entries created
- Projects linked: "Project Alpha", "Personal Care"
```

**Error Response:**
```
⚠️ Calendar Sync Completed with Issues
- 3 events processed
- 1 new task created
- 1 failed (no matching project for "Unknown Meeting")
- 1 duplicate skipped

Would you like me to:
1. Create "Unknown Meeting" as new project?
2. Link to existing project manually?
```

### MVP Implementation Requirements

**Prerequisites:**
1. Get Tasks database schema and ID
2. Get Timesheet database schema and ID  
3. Add "Event ID" property to Tasks database
4. Test MCP tool access to Notion workspace

**Core Functions:**
1. Parse multiple input formats (CSV, structured text)
2. Generate consistent Event IDs for deduplication
3. Fuzzy project matching with fallbacks
4. Create linked Tasks and Timesheet entries
5. Comprehensive error reporting

### Testing Strategy

**Test Data Sets:**
1. Single event (basic functionality)
2. Multiple events same day (batch processing)
3. Events with existing projects (matching)
4. Events needing new projects (creation)
5. Duplicate events (deduplication)
6. Malformed input (error handling)

**Success Criteria:**
- Parse 90%+ of common calendar formats
- Create properly linked database entries
- Zero duplicate events created
- Clear user feedback on all operations

### Next Phase Enhancements

**Phase 1.2:**
- Bulk date range sync ("sync this week")
- Smart event categorization
- Integration with Day planning workflow
- Project suggestion improvements

**Future Automation:**
- Direct calendar API integration (when available)
- Scheduled automatic sync
- Email trigger system