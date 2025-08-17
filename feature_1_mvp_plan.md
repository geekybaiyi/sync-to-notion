# Feature 1 MVP Plan: MCP-Based Calendar Sync

## Updated Plan Based on Schema Analysis and MCP Framework

### Prerequisites & Dependencies

1. **Notion Database Schemas Required:**
   - **Tasks Database** (ID needed)
   - **Timesheet Database** (ID needed) 
   - **Projects Database** (ID: `15b1eee3-fc62-47ce-8a85-5e41ca832d07` - from Journals schema)
   - **Days Database** (ID: `1d3552d6-4163-4835-9b4e-08e9cb27e235` - from Journals schema)

2. **Available MCP Tools:**
   - Notion API tools (create pages, query databases, etc.)
   - No external CLI dependencies needed

### MVP Scope: Manual Calendar Sync via Chat Interface

#### Phase 1.1: Core Sync Functionality

**User Interaction Pattern:**
```
User: "Sync these calendar events to Notion:"
[CSV data or structured text]

Agent Response:
- Parses calendar data
- Creates Tasks and Timesheet entries
- Links to existing Projects
- Reports sync results
```

#### Detailed Workflow

**Step 1: Data Input Format**
- User provides calendar events in simple structured format
- **Recommended format:**
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

**Step 2: Duplicate Prevention Strategy**
- **Event ID Generation:** `hash(date + start_time + title)`
- **Check existing Tasks:** Search Tasks database for matching Event ID
- **Skip if exists:** Prevent duplicate creation

**Step 3: Project Matching Logic**
1. **Search Projects database** using event title keywords
2. **Fuzzy matching:** Extract key terms from event title
3. **Fallback options:**
   - Create "Unassigned" project if none exists
   - Prompt user for manual project assignment
   - Default to "Personal" or "Work" project based on calendar source

**Step 4: Notion Operations (Using MCP Tools)**

For each unique event:

1. **Query Projects Database:**
   ```
   Tool: post-database-query
   Database ID: 15b1eee3-fc62-47ce-8a85-5e41ca832d07
   Filter: title contains event keywords
   ```

2. **Create Task Entry:**
   ```
   Tool: post-page
   Parent: Tasks Database
   Properties:
   - Title: [Event title]
   - Date: [Event date] 
   - Project: [Linked project]
   - Event ID: [Generated hash]
   - Type: "Meeting" or "Appointment"
   ```

3. **Create Timesheet Entry:**
   ```
   Tool: post-page  
   Parent: Timesheet Database
   Properties:
   - Task: [Link to created task]
   - Start Time: [Event start]
   - End Time: [Event end]
   - Date: [Event date]
   ```

4. **Link to Day Entry (if exists):**
   - Query Days database for matching date
   - Link Task to Day if found

### Missing Database Schemas Needed

To complete the MVP, we need the full schemas for:

1. **Tasks Database:**
   - Required properties: Title, Date, Project (relation), Event ID, Type
   - Optional: Status, Priority, Notes

2. **Timesheet Database:**
   - Required properties: Task (relation), Start Time, End Time, Date
   - Optional: Duration (calculated), Notes

### Error Handling Strategy

1. **Invalid Input Format:** Provide clear format examples
2. **Project Matching Failure:** Create fallback project or ask user
3. **Notion API Errors:** Retry logic and user notification
4. **Duplicate Events:** Skip and report count

### Success Metrics for MVP

1. **Functional Success:**
   - Parse 90%+ of common calendar event formats
   - Successfully create linked Tasks and Timesheet entries
   - Prevent duplicate entries

2. **User Experience:**
   - Single-command sync operation
   - Clear success/failure reporting
   - Minimal user intervention required

### Next Phase Considerations

**Phase 1.2 Enhancements:**
- Bulk date range sync ("sync this week's calendar")
- Smart project detection improvements
- Event type categorization (Meeting, Appointment, Block time)
- Integration with existing Day planning workflow

**Phase 2 Automation:**
- Scheduled automatic sync
- Email-based trigger system
- Integration with calendar APIs (when available)

### Technical Implementation Notes

**MCP Tool Chain:**
1. `post-database-query` → Find existing projects/tasks
2. `post-page` → Create new Tasks and Timesheet entries  
3. `patch-page` → Update existing entries if needed
4. Error handling through try/catch patterns

**Data Flow:**
```
User Input → Parse Events → Check Duplicates → Match Projects → Create Tasks → Create Timesheet → Report Results
```

### Required Next Steps

1. **Get missing database schemas** for Tasks and Timesheet
2. **Define exact property mappings** for each database
3. **Create test dataset** with sample calendar events
4. **Build prototype** using MCP tools only
5. **Test with real Notion workspace**