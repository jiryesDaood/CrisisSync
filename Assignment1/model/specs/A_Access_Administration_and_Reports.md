# CrisisSync – Use Case Specifications: Access, Administration and Reports

Use cases: UC-01 to UC-06 and UC-34 to UC-36.
Each specification follows the Visual Paradigm 18.1 *Use Case Details* layout:
Overview → Details tab (pre/post-conditions) → Flow of Events tab (basic flow) → Extensions pane (alternative and exception flows, each branching from a numbered step).

---

## UC-01 Log In

### Overview
- ID: UC-01
- Primary Actor: User (specialised by System Administrator, Logistics and Resources Officer, Dispatcher, Field Commander, Field Crew Member)
- Supporting Actors: None
- Description: A user authenticates with a username and password and receives the interface of their role.
- Trigger: The user starts the CrisisSync client on a workstation.
- Relationships: None. Every other actor-initiated use case requires this one as a pre-condition.
- Requirements: NFR-05.

### Details tab
Pre-conditions:
- The CrisisSync client is installed on the user's workstation.

Post-conditions (Success):
- An authenticated session exists and the main screen of the user's role is displayed.
- The login is recorded in the audit log.

Post-conditions (Failure):
- No session is created and the login screen remains displayed.

### Flow of Events tab – Basic Flow
1. The user enters a username and password and selects "Log In".
2. The client sends the credentials to the server over an encrypted connection.
3. The system verifies the credentials against the stored password hash.
4. The system verifies that the account is active.
5. The system creates a session and records the login in the audit log.
6. The system displays the main screen of the user's role.

### Extensions pane
3a. Exception – Invalid username or password
1. The system displays "Invalid username or password." without revealing which one is wrong.
2. The system records the failed attempt; the flow resumes at step 1.

4a. Exception – Account deactivated
1. The system displays "Your account is inactive. Contact the System Administrator."; the use case ends.

---

## UC-02 Manage User Accounts

### Overview
- ID: UC-02
- Primary Actor: System Administrator
- Supporting Actors: None
- Description: The System Administrator creates, edits, deactivates and reactivates user accounts and assigns each account one of the five roles.
- Trigger: A staff member joins, leaves or changes role.
- Relationships: None

### Details tab
Pre-conditions:
- The System Administrator is logged in.

Post-conditions (Success):
- The account exists with the requested data, role and active state; the change is recorded in the audit log.
- For a Field Commander or Field Crew Member account, the account is linked to the matching crew member in the resource pool.

Post-conditions (Failure):
- The account data is unchanged.

### Flow of Events tab – Basic Flow
1. The Administrator opens "User Accounts".
2. The system displays all accounts with username, name, role and state.
3. The Administrator selects "New Account", or selects an account and chooses "Edit" or "Deactivate".
4. The Administrator enters or changes the username, name, role and initial password.
5. The Administrator saves.
6. The system validates that the username is unique.
7. The system saves the account, stores the password as a hash, and links Field Commander and Field Crew Member accounts to their crew records.
8. The system records the change in the audit log and displays the updated list.

### Extensions pane
3a. Alternative – Deactivate or reactivate an account
1. The system asks for confirmation.
2. The Administrator confirms; the system changes the account state; the flow resumes at step 8.

6a. Exception – Username already exists
1. The system displays "Username already in use."; the flow resumes at step 4.

---

## UC-03 Load Initial Data

### Overview
- ID: UC-03
- Primary Actor: System Administrator
- Supporting Actors: None
- Description: The System Administrator runs the data-loading utility, which imports initial users, crew members and equipment items from a data file (SQL script, JSON or CSV) into the database.
- Trigger: The system is installed or a demonstration environment is prepared.
- Relationships: None

### Details tab
Pre-conditions:
- The System Administrator is logged in.
- A data file is accessible from the workstation.

Post-conditions (Success):
- All records of the file are stored in the database and a summary (records loaded per entity) is displayed.
- The load is recorded in the audit log.

Post-conditions (Failure):
- No record of the file is stored (the load is one transaction) and the errors are listed.

### Flow of Events tab – Basic Flow
1. The Administrator selects "Load Initial Data".
2. The system asks for the data file.
3. The Administrator selects the file.
4. The system detects the format and validates every record (mandatory fields, roles and equipment types from the closed lists, unique IDs).
5. The system inserts all records in one transaction.
6. The system records the load in the audit log and displays the number of users, crew members and equipment items loaded.

### Extensions pane
4a. Exception – Unsupported format or validation errors
1. The system displays the list of errors with line numbers; nothing is stored and the use case ends.

5a. Exception – A record ID already exists in the database
1. The system rolls back the transaction, lists the duplicate IDs and the use case ends.

---

## UC-04 View Incident History

### Overview
- ID: UC-04
- Primary Actor: System Administrator
- Supporting Actors: None
- Description: The System Administrator views the read-only list of incidents (Open, In Settlement, Closed and Cancelled) and filters it.
- Trigger: The Administrator selects "Incident History".
- Relationships: Extended by UC-05 View Incident Details and Timeline.

### Details tab
Pre-conditions:
- The System Administrator is logged in.

Post-conditions (Success):
- The Administrator sees the incidents matching the filter. No data is modified.

Post-conditions (Failure):
- No list is displayed.

### Flow of Events tab – Basic Flow
1. The Administrator selects "Incident History".
2. The system displays all incidents with name, type, status, opening time and closing or cancellation time, newest first.
3. The Administrator sets a filter by date range, incident type and/or status.
4. The system displays the incidents that match the filter.
5. The Administrator closes the history screen.

### Extensions pane
3a. Exception – Invalid date range
1. The system displays "The start date must be before the end date."; the flow resumes at step 3.

4a. Alternative – No incidents match
1. The system displays "No incidents match the filter."; the flow resumes at step 3.

4b. Alternative – Administrator opens an incident
1. UC-05 View Incident Details and Timeline is performed; the flow resumes at step 4.

---

## UC-05 View Incident Details and Timeline (extension)

### Overview
- ID: UC-05
- Primary Actor: System Administrator (through the extended use case UC-04)
- Supporting Actors: None
- Description: The Administrator drills into an incident to see its tasks, the resources reserved for each task, settlement approvals and the full timeline of actions.
- Trigger: The Administrator selects an incident in the history list.
- Relationships: Extends UC-04 View Incident History. Condition: the Administrator selects an incident.

### Details tab
Pre-conditions:
- UC-04 is displaying a list that contains the selected incident.

Post-conditions (Success):
- The incident details and timeline are displayed read-only.

Post-conditions (Failure):
- Not applicable.

### Flow of Events tab – Basic Flow
1. The system displays the incident data and its tasks with their final states.
2. The system displays, for each task, the resources that were reserved for it and the reservation period.
3. The system displays the settlement approval and transaction token, if any.
4. The system displays the chronological timeline of all audit entries for the incident with timestamp and user.
5. The Administrator returns to the history list.

### Extensions pane
3a. Alternative – No settlement exists (incident Open, In Settlement or Cancelled)
1. The system displays "No settlement"; the flow continues at step 4.

---

## UC-06 View Management Reports

### Overview
- ID: UC-06
- Primary Actor: System Administrator
- Supporting Actors: None
- Description: The System Administrator views stored monthly reports (Resource Utilization and SLA Compliance).
- Trigger: The Administrator selects "Reports".
- Relationships: None. Reports are produced by UC-34, UC-35 and UC-36.
- Requirements: FR-19, NFR-06.

### Details tab
Pre-conditions:
- The System Administrator is logged in.

Post-conditions (Success):
- The selected report is displayed.

Post-conditions (Failure):
- No report is displayed.

### Flow of Events tab – Basic Flow
1. The Administrator selects "Reports".
2. The system verifies that the user has the System Administrator role and displays the stored reports grouped by month and report type.
3. The Administrator selects a report.
4. The system displays the report in tabular and graphical form.

### Extensions pane
2a. Alternative – No stored reports
1. The system displays "No reports have been generated yet." and offers on-demand generation (UC-35, UC-36); the use case ends.

2b. Exception – The request comes from a session without the System Administrator role
1. The server rejects the request and returns no report data; the Reports menu is not displayed to other roles.
2. The system records the rejected attempt in the audit log within 1 second; the use case ends.

---

## UC-34 Generate Monthly Reports

### Overview
- ID: UC-34
- Primary Actor: System Clock
- Supporting Actors: None
- Description: At the end of every calendar month the system clock triggers automatic generation of all monthly management reports.
- Trigger: The server clock reaches 00:05 on the first day of a month.
- Relationships: Includes UC-35 Generate SLA Compliance Report; includes UC-36 Generate Resource Utilization Report.
- Requirements: FR-14, NFR-10.

### Details tab
Pre-conditions:
- None.

Post-conditions (Success):
- The SLA Compliance Report and the Resource Utilization Report for the previous month are stored and visible in UC-06.
- The generation is recorded in the audit log.

Post-conditions (Failure):
- No report for the month is stored, and the failure is recorded in the audit log.

### Flow of Events tab – Basic Flow
1. The System Clock signals the start of a new month.
2. The system determines the report month (the previous calendar month).
3. The system generates the SLA Compliance Report (include UC-35).
4. The system generates the Resource Utilization Report (include UC-36).
5. The system stores both reports, records the generation in the audit log and marks them as new for the System Administrator.

### Extensions pane
2a. Alternative – Reports for the month already exist
1. The system does not generate them again; the use case ends.

---

## UC-35 Generate SLA Compliance Report

### Overview
- ID: UC-35
- Primary Actor: System Administrator (on demand for a past month); also included by UC-34 for the automatic monthly run
- Supporting Actors: None
- Description: The system computes, per incident type, the number of incidents finally closed in one calendar month and the mean time from incident opening to the last Completed task; cancelled tasks are ignored (D-07, D-08).
- Trigger: The Administrator requests the report for a selected past month, or UC-34 runs.
- Relationships: Included by UC-34 Generate Monthly Reports.
- Requirements: FR-15, FR-16, FR-17, FR-18, NFR-04, NFR-06, NFR-12.

### Details tab
Pre-conditions:
- For the on-demand run: the System Administrator is logged in.

Post-conditions (Success):
- The SLA Compliance Report for the month is stored and displayed (on demand) or stored (automatic run).

Post-conditions (Failure):
- No report is stored and the reason is shown to the Administrator or recorded in the audit log.

### Flow of Events tab – Basic Flow
1. The Administrator selects "Reports", then "Generate SLA Compliance Report", selects a month and selects "Generate".
2. The system verifies that the user has the System Administrator role and that the month is a completed past month.
3. The system selects the qualifying incidents: status Closed, closing timestamp inside the month, and at least one Completed task.
4. For each qualifying incident, the system computes the time from the incident opening to the last Completed task; cancelled tasks are ignored.
5. For each of the 5 incident types, the system computes the number of qualifying incidents and the mean time (hh:mm).
6. The system stores the report with its month and generation timestamp.
7. The system displays the report as a table with one row per incident type.

### Extensions pane
1a. Alternative – Automatic run (included by UC-34)
1. The month is supplied by UC-34; steps 1, 2 and 7 are skipped.

2a. Exception – Current or future month selected
1. The system displays "Reports can be generated only for completed months."; the flow resumes at step 1.

2b. Exception – The request comes from a session without the System Administrator role
1. The server rejects the request, stores no report and returns no report data; the Reports menu is not displayed to other roles.
2. The system records the rejected attempt in the audit log within 1 second; the use case ends.

5a. Alternative – No qualifying incidents of a type
1. The system shows count 0 and mean "N/A" for that type; the flow continues at step 6.

---

## UC-36 Generate Resource Utilization Report

### Overview
- ID: UC-36
- Primary Actor: System Administrator (on demand for a past month); also included by UC-34 for the automatic monthly run
- Supporting Actors: None
- Description: The system computes, for one calendar month, the utilization percentage of equipment and personnel (share of the month's time each resource was Reserved), broken down by equipment type and crew role, and shows it graphically.
- Trigger: The Administrator requests the report for a selected past month, or UC-34 runs.
- Relationships: Included by UC-34 Generate Monthly Reports.

### Details tab
Pre-conditions:
- For the on-demand run: the System Administrator is logged in.

Post-conditions (Success):
- The Resource Utilization Report for the month is stored and displayed (on demand) or stored (automatic run).

Post-conditions (Failure):
- No report is stored and the reason is shown or recorded in the audit log.

### Flow of Events tab – Basic Flow
1. The Administrator selects "Reports", then "Generate Resource Utilization Report", selects a month and selects "Generate".
2. The system verifies that the user has the System Administrator role and that the month is a completed past month.
3. For each resource, the system sums the time of its assignment records (from reservation to release, including records closed early when the resource was released during a task) that falls in the month and divides it by the length of the month.
4. The system averages the percentages per equipment type and per crew role.
5. The system stores the report.
6. The system displays the report as a bar chart and a table per equipment type and per role.

### Extensions pane
1a. Alternative – Automatic run (included by UC-34)
1. The month is supplied by UC-34; steps 1, 2 and 6 are skipped.

2a. Exception – Current or future month selected
1. The system displays "Reports can be generated only for completed months."; the flow resumes at step 1.

2b. Exception – The request comes from a session without the System Administrator role
1. The server rejects the request, stores no report and returns no report data; the Reports menu is not displayed to other roles.
2. The system records the rejected attempt in the audit log within 1 second; the use case ends.

3a. Alternative – Reservation spans the month boundary
1. The system counts only the part of the reservation inside the month.
