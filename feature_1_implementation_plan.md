# Feature 1 Implementation Plan: Calendar Sync with Exact Database Schemas

## Database Schema Analysis

### Key Databases for Calendar Sync

**Tasks Database** (ID: `eec0f1cd-7b83-40fe-a558-17b106406ef4`)
- ✅ **Event ID**: `rich_text` - Already exists! (Line 240)
- ✅ **Name**: `title` - Task title
- ✅ **Do Time**: `date` - When to do the task
- ✅ **Project**: `relation` to Projects DB (ID: `15b1eee3-fc62-47ce-8a85-5e41ca832d07`)
- ✅ **Type**: `multi_select` with options including "Discussion/Meeting"
- ✅ **Status**: `select` - Task status
- ✅ **Summary**: `rich_text` - Additional details

**Time Sheet Database** (ID: `13540fc4-bdd0-8085-8b33-f0a8deac55f9`)
- ✅ **Name**: `title` - Entry title
- ✅ **Start**: `date` - Start time
- ✅ **End**: `date` - End time  
- ✅ **Task**: `relation` to Tasks DB
- ✅ **Day**: `relation` to Daily Logs DB (ID: `1d3552d6-4163-4835-9b4e-08e9cb27e235`)
- ✅ **Summary**: `rich_text` - Entry details

**Projects Database** (ID: `15b1eee3-fc62-47ce-8a85-5e41ca832d07`)
- ✅ **Name**: `title` - Project name
- ✅ **Status**: `select` - Project status
- ✅ **AOF**: `multi_select` - Areas of Focus

## Implementation Specification

### Phase 1.1: MCP-Based Calendar Sync

#### Input Format Options

**Option 1: Structured Text**
```
Event: Project Alpha Kickoff
Date: 2025-01-17
Time: 10:00-11:00
Calendar: Work
Description: Initial project meeting

Event: Dentist Appointment
Date: 2025-01-17
Time: 14:00-15:30
Calendar: Personal
```

**Option 2: CSV Format**
```csv
title,date,start_time,end_time,calendar_source,description
"Project Alpha Kickoff","2025-01-17","10:00","11:00","Work","Initial project meeting"
"Dentist Appointment","2025-01-17","14:00","15:30","Personal","Routine checkup"
```

#### Step-by-Step MCP Workflow

**Step 1: Parse and Validate Input**
- Extract event details from user input
- Generate Event ID: `SHA256(date + start_time + title)[0:8]`
- Validate date/time formats

**Step 2: Check for Duplicates**
```json
{
  "tool": "post-database-query",
  "database_id": "eec0f1cd-7b83-40fe-a558-17b106406ef4",
  "filter": {
    "property": "Event ID",
    "rich_text": {
      "equals": "generated_event_id"
    }
  }
}
```

**Step 3: Find or Create Project**
```json
{
  "tool": "post-database-query", 
  "database_id": "15b1eee3-fc62-47ce-8a85-5e41ca832d07",
  "filter": {
    "property": "Name",
    "title": {
      "contains": "extracted_keywords"
    }
  }
}
```

**Fallback Project Creation:**
```json
{
  "tool": "post-page",
  "parent": {
    "database_id": "15b1eee3-fc62-47ce-8a85-5e41ca832d07"
  },
  "properties": {
    "Name": {
      "title": [{"text": {"content": "Calendar Events - {calendar_source}"}}]
    },
    "Status": {
      "select": {"name": "Active"}
    },
    "AOF": {
      "multi_select": [{"name": "05-Career"}]
    }
  }
}
```

**Step 4: Create Task Entry**
```json
{
  "tool": "post-page",
  "parent": {
    "database_id": "eec0f1cd-7b83-40fe-a558-17b106406ef4"
  },
  "properties": {
    "Name": {
      "title": [{"text": {"content": "event_title"}}]
    },
    "Event ID": {
      "rich_text": [{"text": {"content": "generated_event_id"}}]
    },
    "Do Time": {
      "date": {"start": "2025-01-17T10:00:00"}
    },
    "Project": {
      "relation": [{"id": "project_page_id"}]
    },
    "Type": {
      "multi_select": [{"name": "Discussion/Meeting"}]
    },
    "Status": {
      "select": {"name": "In Progress"}
    },
    "Summary": {
      "rich_text": [{"text": {"content": "Imported from calendar: description"}}]
    }
  }
}
```

**Step 5: Create Time Sheet Entry**
```json
{
  "tool": "post-page",
  "parent": {
    "database_id": "13540fc4-bdd0-8085-8b33-f0a8deac55f9"
  },
  "properties": {
    "Name": {
      "title": [{"text": {"content": "event_title"}}]
    },
    "Start": {
      "date": {"start": "2025-01-17T10:00:00"}
    },
    "End": {
      "date": {"start": "2025-01-17T11:00:00"}
    },
    "Task": {
      "relation": [{"id": "created_task_page_id"}]
    },
    "Summary": {
      "rich_text": [{"text": {"content": "Calendar event: description"}}]
    }
  }
}
```

**Step 6: Link to Daily Log (Optional)**
```json
{
  "tool": "post-database-query",
  "database_id": "1d3552d6-4163-4835-9b4e-08e9cb27e235",
  "filter": {
    "property": "Date",
    "date": {
      "equals": "2025-01-17"
    }
  }
}
```

#### Event Type Classification

**Meeting/Discussion Events:**
- Keywords: "meeting", "call", "sync", "standup", "review"
- Type: "Discussion/Meeting"
- AOF: "05-Career" (Work) or "04-Spouse" (Personal)

**Appointments:**
- Keywords: "appointment", "doctor", "dentist", "therapy"
- Type: "Dr. Appt"
- AOF: "01-Health"

**Exercise/Fitness:**
- Keywords: "gym", "workout", "run", "yoga", "exercise"
- Type: "Exercise - 有氧"
- AOF: "01-Health"

**Personal/Social:**
- Keywords: "dinner", "lunch", "hangout", "family"
- Type: "Hangout"
- AOF: "04-Spouse", "06-Family", or "08-Friends"

#### Project Matching Algorithm

**1. Exact Match:** Search project names for exact title match
**2. Keyword Match:** Extract 2-3 key words from event title, search projects
**3. Calendar-based Fallback:**
   - Work calendar → "Work Projects" or "Calendar Events - Work"
   - Personal calendar → "Personal Tasks" or "Calendar Events - Personal"
**4. Create New Project:** If no match and unique event pattern

#### Error Handling Strategy

**Duplicate Events:**
```
✅ Event already exists: "Project Alpha Kickoff" (Event ID: abc12345)
Skipping duplicate entry.
```

**Project Matching Failure:**
```
⚠️ No matching project found for "Unknown Meeting"
Created new project: "Calendar Events - Work"
```

**API Errors:**
```
❌ Failed to create task: API rate limit exceeded
Retrying in 1 second... (Attempt 2/3)
```

#### Success Response Format

```
✅ Calendar Sync Complete - January 17, 2025

📊 Summary:
• 4 events processed
• 3 new tasks created
• 1 duplicate skipped
• 3 timesheet entries created

📋 Tasks Created:
• "Project Alpha Kickoff" → Project: Work Projects
• "Dentist Appointment" → Project: Health & Wellness  
• "Team Lunch" → Project: Calendar Events - Work

🔗 Projects Used:
• Work Projects (2 events)
• Health & Wellness (1 event)

⏰ Time Tracking:
• Total scheduled time: 3.5 hours
• Work: 2.0 hours
• Personal: 1.5 hours
```

### Testing Scenarios

**Test Case 1: Basic Single Event**
```
Input: "Meeting with John, 2025-01-17, 10:00-11:00, Work"
Expected: 1 task, 1 timesheet, project matched/created
```

**Test Case 2: Multiple Events Same Day**
```
Input: CSV with 5 events on same day
Expected: 5 tasks, 5 timesheet entries, proper project linking
```

**Test Case 3: Duplicate Prevention**
```
Input: Same event twice in different formats
Expected: 1 task created, 1 duplicate skipped
```

**Test Case 4: Unknown Project**
```
Input: Event with title not matching any project
Expected: New fallback project created
```

### Next Phase Enhancements

**Phase 1.2: Bulk Operations**
- Date range sync: "Sync calendar for this week"
- Recurring event detection
- Smart event categorization improvements

**Phase 1.3: Integration**
- Link with Daily Log planning workflow
- Automatic AOF assignment based on calendar patterns
- Time blocking optimization

### Required Database Modifications

**Tasks Database:**
- ✅ Event ID property already exists
- No modifications needed

**Projects Database:**
- Consider adding "Calendar Source" property for auto-created projects
- Optional: "Auto-Created" checkbox to identify system-generated projects

### Implementation Priority

1. **Core Sync Functionality** (Week 1)
   - Basic parsing and task creation
   - Simple project matching
   - Duplicate prevention

2. **Enhanced Matching** (Week 2)  
   - Smart project matching algorithm
   - Event type classification
   - Fallback project creation

3. **User Experience** (Week 3)
   - Rich success/error reporting
   - Bulk operation support
   - Input format flexibility

4. **Integration** (Week 4)
   - Daily Log linking
   - Time tracking optimization
   - AOF-based categorization