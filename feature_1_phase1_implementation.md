# Feature 1 Phase 1: Implementation Plan - Calendar to Tasks Only

## Implementation Overview

**Scope**: CSV calendar events → Notion Tasks (with project linking, no timesheet creation)

**Key Changes**:
- ❌ No timesheet entry creation
- ❌ No new project creation (report only)
- ✅ CSV with project column for easier matching
- ✅ Database caching for performance
- ✅ Simplified task-only workflow

## Database Caching Strategy

### Initial Cache Load
```json
{
  "cache_initialization": {
    "projects": {
      "database_id": "15b1eee3-fc62-47ce-8a85-5e41ca832d07",
      "query": {
        "filter": {
          "and": [
            {
              "property": "Status",
              "select": {
                "does_not_equal": "Completed"
              }
            },
            {
              "property": "Status", 
              "select": {
                "does_not_equal": "Pause"
              }
            }
          ]
        },
        "page_size": 100
      },
      "properties": ["Name", "Status", "AOF"]
    },
    "goals": {
      "database_id": "bf0e1fef-e975-4cf8-89d6-0cd9bed9aa32",
      "query": {
        "filter": {
          "property": "Status",
          "select": {
            "does_not_equal": "Achieved"
          }
        }
      },
      "properties": ["Name", "Status", "AOF"]
    },
    "current_month": {
      "database_id": "4ee6df88-6558-48b4-862d-2aaa1328ccd0",
      "query": {
        "filter": {
          "property": "isCurrent",
          "formula": {
            "checkbox": {
              "equals": true
            }
          }
        }
      }
    },
    "current_week": {
      "database_id": "8ab06911-3877-4a07-8e75-040c46c560e5", 
      "query": {
        "filter": {
          "property": "isCurrent",
          "formula": {
            "checkbox": {
              "equals": true
            }
          }
        }
      }
    }
  }
}
```

## Updated CSV Input Format

### Required Columns:
```csv
title,date,start_time,end_time,calendar_source,project,description
"Sprint Planning","2025-01-17","10:00","11:00","Work","Development","Sprint 3 kickoff"
"Doctor Checkup","2025-01-18","14:00","15:30","Personal","Health","Annual physical"
"Client Call","2025-01-19","10:00","11:00","Work","","Weekly sync call"
```

### Column Definitions:
- **title**: Event name → Task Name
- **date**: Event date → Task Do Time
- **start_time**: Event start → Used for Do Time 
- **end_time**: Event end → (Ignored in Phase 1, no timesheet)
- **calendar_source**: "Work"/"Personal" → Used for Type classification
- **project**: Project hint for matching → Used to find existing project
- **description**: Event details → Task Summary

## Implementation Workflow

### Step 1: Cache Initialization
```python
def initialize_cache():
    """Load reference databases for fast lookup"""
    cache = {}
    
    # Load active projects
    cache['projects'] = query_database(
        database_id="15b1eee3-fc62-47ce-8a85-5e41ca832d07",
        filter={"Status": {"select": {"does_not_equal": "Completed"}}}
    )
    
    # Load active goals  
    cache['goals'] = query_database(
        database_id="bf0e1fef-e975-4cf8-89d6-0cd9bed9aa32",
        filter={"Status": {"select": {"does_not_equal": "Achieved"}}}
    )
    
    # Load current time periods
    cache['current_month'] = query_database(
        database_id="4ee6df88-6558-48b4-862d-2aaa1328ccd0",
        filter={"isCurrent": {"formula": {"checkbox": {"equals": true}}}}
    )
    
    return cache
```

### Step 2: CSV Parsing
```python
def parse_csv_input(csv_text):
    """Parse CSV and extract event data"""
    events = []
    
    for row in csv.DictReader(io.StringIO(csv_text)):
        event = {
            'title': row['title'],
            'date': row['date'], 
            'start_time': row['start_time'],
            'end_time': row['end_time'],  # Store but don't use in Phase 1
            'calendar_source': row['calendar_source'],
            'project_hint': row.get('project', ''),
            'description': row.get('description', ''),
            'event_id': generate_event_id(row['date'], row['start_time'], row['title'])
        }
        events.append(event)
    
    return events
```

### Step 3: Project Matching (No Creation)
```python
def match_project(project_hint, event_title, cache):
    """Find existing project, don't create new ones"""
    
    if project_hint:
        # Try exact project hint match first
        for project in cache['projects']:
            if project_hint.lower() in project['Name'].lower():
                return project, f"Matched via project hint: '{project_hint}'"
    
    # Try auto-matching with event title
    keywords = extract_keywords(event_title)
    for project in cache['projects']:
        for keyword in keywords:
            if keyword.lower() in project['Name'].lower():
                return project, f"Auto-matched via keyword: '{keyword}'"
    
    # No match found
    return None, f"No project found for hint: '{project_hint}'" if project_hint else "No project auto-matched"
```

### Step 4: Task Creation (Only)
```python
def create_task(event, project, cache):
    """Create single task entry - no timesheet"""
    
    # Determine task type
    task_type = classify_event_type(event['title'], event['calendar_source'])
    
    # Build task properties
    properties = {
        "Name": {
            "title": [{"text": {"content": event['title']}}]
        },
        "Event ID": {
            "rich_text": [{"text": {"content": event['event_id']}}]
        },
        "Do Time": {
            "date": {"start": f"{event['date']}T{event['start_time']}:00"}
        },
        "Type": {
            "multi_select": [{"name": task_type}]
        },
        "Status": {
            "select": {"name": "In Progress"}
        }
    }
    
    # Add project link if found
    if project:
        properties["Project"] = {
            "relation": [{"id": project['id']}]
        }
    
    # Add description if provided
    if event['description']:
        properties["Summary"] = {
            "rich_text": [{"text": {"content": f"Calendar event: {event['description']}"}}]
        }
    
    # Create task via MCP
    return create_page(
        parent={"database_id": "eec0f1cd-7b83-40fe-a558-17b106406ef4"},
        properties=properties
    )
```

## MCP Tool Calls Sequence

### 1. Initialize Cache (Session Start)
```json
[
  {
    "tool": "post-database-query",
    "database_id": "15b1eee3-fc62-47ce-8a85-5e41ca832d07",
    "filter": {
      "property": "Status",
      "select": {"does_not_equal": "Completed"}
    },
    "page_size": 100
  },
  {
    "tool": "post-database-query", 
    "database_id": "bf0e1fef-e975-4cf8-89d6-0cd9bed9aa32",
    "filter": {
      "property": "Status",
      "select": {"does_not_equal": "Achieved"}
    }
  }
]
```

### 2. Check for Duplicates (Per Event)
```json
{
  "tool": "post-database-query",
  "database_id": "eec0f1cd-7b83-40fe-a558-17b106406ef4",
  "filter": {
    "property": "Event ID",
    "rich_text": {
      "equals": "generated_event_id_abc123"
    }
  }
}
```

### 3. Create Task (Per Non-Duplicate Event)
```json
{
  "tool": "post-page",
  "parent": {
    "database_id": "eec0f1cd-7b83-40fe-a558-17b106406ef4"
  },
  "properties": {
    "Name": {
      "title": [{"text": {"content": "Sprint Planning"}}]
    },
    "Event ID": {
      "rich_text": [{"text": {"content": "abc123ef"}}]
    },
    "Do Time": {
      "date": {"start": "2025-01-17T10:00:00"}
    },
    "Project": {
      "relation": [{"id": "project_page_id_if_found"}]
    },
    "Type": {
      "multi_select": [{"name": "Discussion/Meeting"}]
    },
    "Status": {
      "select": {"name": "In Progress"}
    },
    "Summary": {
      "rich_text": [{"text": {"content": "Calendar event: Sprint 3 kickoff"}}]
    }
  }
}
```

## Event Type Classification

### Classification Rules:
```python
def classify_event_type(title, calendar_source):
    """Determine task type from event details"""
    
    title_lower = title.lower()
    
    # Meeting types
    if any(word in title_lower for word in ['meeting', 'sync', 'standup', 'call']):
        return "Discussion/Meeting"
    
    # Planning types  
    if any(word in title_lower for word in ['planning', 'review', 'retrospective']):
        return "Planning & Review"
    
    # Health appointments
    if any(word in title_lower for word in ['doctor', 'dentist', 'appointment', 'checkup']):
        return "Dr. Appt"
    
    # Exercise
    if any(word in title_lower for word in ['gym', 'workout', 'exercise', 'run']):
        return "Exercise - 有氧"
    
    # Social/Personal
    if any(word in title_lower for word in ['lunch', 'dinner', 'hangout']):
        return "Hangout"
    
    # Default based on calendar
    if calendar_source == "Personal":
        return "Routine - Personal"
    else:
        return "Discussion/Meeting"
```

## Response Format

### Successful Sync Response:
```
✅ Calendar Sync Complete - January 17, 2025

🔄 Cache Status:
• 23 active projects loaded
• 8 active goals loaded
• Current month: January 2025
• Current week: Week 3

📊 Processing Summary:
• 4 events in CSV
• 3 new tasks created
• 1 duplicate skipped

📋 Tasks Created:
• "Sprint Planning" 
  ✅ Project: Software Development Project
  📅 Date: Jan 17, 10:00 AM
  
• "Doctor Checkup" 
  ✅ Project: Health & Wellness
  📅 Date: Jan 18, 2:00 PM
  
• "Client Call"
  ❌ No project linked (auto-match failed)
  📅 Date: Jan 19, 10:00 AM

⚠️ Projects Not Found:
• No project hint provided for "Client Call"

💡 Next Steps:
• Review unlinked tasks in Notion
• Add project links manually if needed
• Consider updating project hints in future CSVs
```

## Error Scenarios

### Project Not Found:
```
Event: "Team Meeting" with project hint: "NonExistent"
→ Search cache for "NonExistent" 
→ No matches found
→ Create task WITHOUT project link
→ Report in final summary: "Project not found: NonExistent"
```

### Duplicate Event:
```
Event: Same title + date + time as existing task
→ Check Event ID against Tasks database
→ Match found: Skip creation
→ Report: "Duplicate skipped: Sprint Planning (Event ID: abc123)"
```

### Invalid CSV:
```
Missing required columns: title, date, start_time
→ Validation error before processing
→ Return format examples and requirements
→ Request corrected CSV
```

## Performance Metrics

### Target Performance:
- **Cache Load**: < 2 seconds for 100 projects
- **Event Processing**: < 0.5 seconds per event
- **Total Sync**: < 5 seconds for 10 events
- **Memory Usage**: < 50MB for cached data

### API Call Optimization:
- **Cache Load**: 4 initial queries (vs N queries per event)
- **Duplicate Check**: 1 query per event (unavoidable)
- **Task Creation**: 1 call per new task
- **Total**: ~15 API calls for 10 events (vs 30+ without caching)

## Testing Strategy

### Test Case 1: Perfect Matching
```csv
title,date,start_time,end_time,calendar_source,project,description
"Sprint Planning","2025-01-17","10:00","11:00","Work","Development","Sprint 3"
```
Expected: 1 task created, linked to project containing "Development"

### Test Case 2: Project Not Found
```csv
title,date,start_time,end_time,calendar_source,project,description
"Mystery Meeting","2025-01-17","14:00","15:00","Work","InvalidProject",""
```
Expected: 1 task created, NO project link, reported in summary

### Test Case 3: Auto-Matching (Empty Project)
```csv
title,date,start_time,end_time,calendar_source,project,description
"Health Review Meeting","2025-01-17","16:00","17:00","Personal","",""
```
Expected: 1 task created, auto-matched to project containing "Health"

### Test Case 4: Duplicate Prevention
```csv
# Same event run twice
title,date,start_time,end_time,calendar_source,project,description
"Daily Standup","2025-01-17","09:00","09:30","Work","Development",""
"Daily Standup","2025-01-17","09:00","09:30","Work","Development",""
```
Expected: 1 task created, 1 duplicate skipped

## Phase 2 Readiness

### Architecture Prepared For:
- ✅ **Timesheet Creation**: Event has start/end times stored
- ✅ **Daily Log Linking**: Cache includes current time periods
- ✅ **New Project Creation**: Project matching infrastructure exists
- ✅ **Goal Integration**: Goals cache ready for linking

### Clean Transition:
- Event ID system supports correlation across databases
- Project matching can be extended to creation workflow
- Cache strategy scales to additional databases
- CSV format can be extended with new columns