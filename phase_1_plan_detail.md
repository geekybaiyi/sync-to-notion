# Feature 1 Detailed Plan: On-Demand Calendar Sync

This document provides a detailed breakdown of the plan for Feature 1, starting with a manual, input-driven workflow.

## Phase 1.1: Manual Calendar Export and Sync

This phase relies on the user providing the calendar data directly, removing the need for system-level CLI tools.

### Step 1: User's Manual Export

1.  **User Action:** The user will use their preferred tool (e.g., Gemini) to fetch daily calendar events from all required Google accounts.
2.  **Required Format:** The user will provide the data to the Gemini CLI agent in a structured format. A simple CSV (Comma-Separated Values) format is recommended for clarity and ease of parsing. The user will paste this directly into the chat.

    **Example CSV Format:**
    ```csv
    start_time,end_time,title,calendar_source
    "2025-08-17 10:00","2025-08-17 11:00","Project Alpha Kickoff","Work"
    "2025-08-17 14:00","2025-08-17 15:30","Dentist Appointment","Personal"
    ```

### Step 2: The Sync Process (Agent's Role)

1.  **User Command:** The user will initiate the sync with a command like, "Sync these calendar events to Notion:" followed by the CSV-formatted data.

2.  **Duplicate Prevention (Crucial):**
    *   To prevent creating the same task multiple times, a mechanism to track imported events is essential.
    *   **Proposal:** Add a new **text property** to the "Tasks" database named `Event ID`.
    *   **Mechanism:** A unique ID is generated from each event's `start_time` and `title` from the CSV data. Before creating a task, the agent searches for an existing task with that ID. If found, the event is skipped.
    *   **Permission Required:** This step requires user permission to add a new property to their Notion database.

3.  **Execution Workflow:**
    *   **Parse Input:** The agent will parse the CSV data provided by the user.
    *   **Process Each Event:** For each row in the data (and if the `Event ID` doesn't already exist):
        *   **Find Project:** Search the "Projects" database for a relevant project based on the event `title`.
        *   **Create Task:** Create a new task in the "Tasks" database, linking it to the project and populating the `Event ID` field.
        *   **Create Timesheet Entry:** Create the associated timesheet entry, linking it to the new task and using the `start_time` and `end_time`.
    *   **Report Back:** The agent will confirm when the sync is complete and report how many new events were added.