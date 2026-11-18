# CrisisSync – Use Case Specifications: Resource Pool and Field Operations

Use cases: UC-07 to UC-12 and UC-28.
Each specification follows the Visual Paradigm 18.1 *Use Case Details* layout:
Overview → Details tab (pre/post-conditions) → Flow of Events tab (basic flow) → Extensions pane (alternative and exception flows, each branching from a numbered step).

---

## UC-07 Update Equipment Maintenance Status

### Overview
- ID: UC-07
- Primary Actor: Logistics and Resources Officer
- Supporting Actors: None
- Description: The Logistics and Resources Officer marks an equipment item as Faulty/In Repair or returns a repaired item to Available.
- Trigger: An item is sent to or returns from maintenance.
- Relationships: None

### Details tab
Pre-conditions:
- The Logistics and Resources Officer is logged in.
- The equipment item exists in the resource pool.

Post-conditions (Success):
- The item's status is Faulty/In Repair or Available as requested.
- The update is recorded in the audit log and pushed to all relevant workstations.

Post-conditions (Failure):
- The item's status is unchanged.

### Flow of Events tab – Basic Flow
1. The Logistics and Resources Officer opens the equipment list.
2. The system displays all equipment items with type, ID, current location and status.
3. The Logistics and Resources Officer selects an item and chooses the new status (Faulty/In Repair or Available).
4. The system verifies that the transition is allowed.
5. The system saves the new status.
6. The system records the update in the audit log and pushes the change to all relevant workstations.
7. The system displays the updated item.

### Extensions pane
4a. Exception – Item is Reserved for an Active task
1. The system displays "The item is reserved for task <name>. Faults during a task are reported by the field team (UC-12)."; the use case ends.

5a. Exception – Concurrent update
1. The system detects that the item's status changed since step 2.
2. The system discards the change, displays the current status and the flow resumes at step 3.

---

## UC-08 Transfer Equipment Between Stations

### Overview
- ID: UC-08
- Primary Actor: Logistics and Resources Officer
- Supporting Actors: None
- Description: The Logistics and Resources Officer moves an equipment item from one station to another and the system updates the item's current location.
- Trigger: The Logistics and Resources Officer decides to rebalance equipment between stations.
- Relationships: None

### Details tab
Pre-conditions:
- The Logistics and Resources Officer is logged in.

Post-conditions (Success):
- The item's current location is the destination station; the change is recorded in the audit log.

Post-conditions (Failure):
- The item's location is unchanged.

### Flow of Events tab – Basic Flow
1. The Logistics and Resources Officer selects an equipment item and chooses "Transfer".
2. The system displays the item's current station and the list of stations.
3. The Logistics and Resources Officer selects the destination station and confirms.
4. The system verifies that the item is not Reserved and that the destination differs from the current station.
5. The system updates the item's current location.
6. The system records the transfer in the audit log and pushes the change.

### Extensions pane
4a. Exception – Item is Reserved
1. The system displays "A reserved item cannot be transferred."; the use case ends.

4b. Exception – Destination equals the current station
1. The system displays "Select a different station."; the flow resumes at step 3.

---

## UC-09 Update Crew Member Availability

### Overview
- ID: UC-09
- Primary Actor: Logistics and Resources Officer
- Supporting Actors: None
- Description: The Logistics and Resources Officer records an exceptional absence of a crew member (vacation, sick leave) or ends it, so that the member cannot or can again be reserved. The Logistics and Resources Officer is the only user who can mark a Reserved crew member as unavailable; this releases them from the Active task exactly as for an equipment fault, and the Dispatcher must assign a replacement (D-10).
- Trigger: The Logistics and Resources Officer is informed of a crew member's absence or return.
- Relationships: None. A release from an Active task is usually followed by UC-28 Assign Replacement Resource performed by the Dispatcher.
- Requirements: FR-12.

### Details tab
Pre-conditions:
- The Logistics and Resources Officer is logged in.
- The crew member exists in the resource pool.

Post-conditions (Success):
- The crew member's status is Unavailable (with reason and period) or Available.
- If the member was Reserved for an Active task: they are no longer reserved for it, their assignment record is closed with the release time, the package item they filled is missing, the task stays Active and all Dispatchers have received a "Replacement required" alert.
- The update is recorded in the audit log and pushed to all relevant workstations.

Post-conditions (Failure):
- The crew member's status and any reservation are unchanged.

### Flow of Events tab – Basic Flow
1. The Logistics and Resources Officer opens the personnel list.
2. The system displays all crew members with ID, name, role, current location and status.
3. The Logistics and Resources Officer selects a crew member and chooses "Set Unavailable" with a reason (Vacation or Sick Leave) and a period, or "Set Available".
4. The system checks whether the crew member is Reserved for a task, and verifies that they are not.
5. The system re-verifies that the member is not Reserved and saves the new status.
6. The system records the update in the audit log and pushes the change.

### Extensions pane
3a. Exception – Invalid period
1. The system displays "The end date must be after the start date."; the flow resumes at step 3.

4a. Alternative – Crew member is Reserved for an Active task and the Logistics and Resources Officer chose "Set Unavailable"
1. The system displays "<name> is reserved for task <task>. Setting them Unavailable releases them from the task, and the Dispatcher must assign a replacement." and asks for confirmation.
2. The Logistics and Resources Officer confirms.
3. In one transaction, the system verifies that the member is still Reserved for the task found in step 4. If the assignment changed in the meantime (another operation came first), the system rolls back and the flow resumes at step 4, so the request is checked against the current state.
4. The system sets the member to Unavailable, releases them from the task, closes their assignment record with the release time (the record still counts for costs and utilization) and marks the package item they filled as missing; the task stays Active.
5. The system commits, records the changes in the audit log, pushes them to all relevant workstations and sends all Dispatchers a "Replacement required" alert for the task. If the member was the task's Field Commander, the alert states that a replacement Field Commander is mandatory; the use case ends.

4b. Exception – Crew member is Reserved and the Logistics and Resources Officer chose "Set Available"
1. The system displays "<name> is already working on task <task>."; the use case ends.

5a. Exception – The crew member was reserved concurrently
1. The system finds on re-verification that the member is now Reserved for a task; the flow resumes at step 4.

---

## UC-10 Report Shift Start and Readiness

### Overview
- ID: UC-10
- Primary Actor: Field Crew Member
- Supporting Actors: None
- Description: A Field Crew Member reports the start of a shift and declares readiness (Available or Not Available) and current location. A crew member who is Reserved for a task cannot mark themselves Not Available; only the Logistics and Resources Officer can do that (UC-09, D-10).
- Trigger: The crew member arrives for a shift or their readiness changes.
- Relationships: None

### Details tab
Pre-conditions:
- The Field Crew Member is logged in.

Post-conditions (Success):
- The crew member's status and current location are updated; the change is recorded in the audit log and pushed to the Dispatchers.

Post-conditions (Failure):
- The crew member's status and any reservation are unchanged.

### Flow of Events tab – Basic Flow
1. The Field Crew Member selects "Start Shift".
2. The system displays the current status and asks for readiness and current location.
3. The Field Crew Member selects Available or Not Available and confirms the location.
4. The system checks whether the member is Reserved for a task or has an Unavailable period set by the Logistics and Resources Officer, and verifies that neither is the case.
5. The system re-verifies that the member is not Reserved and has no Unavailable period, updates the member's status and location and records the shift start time.
6. The system records the update in the audit log and pushes it to all Dispatcher workstations.

### Extensions pane
4a. Exception – Member is on recorded leave
1. The system displays "You are registered as unavailable until <date>. Contact the Logistics and Resources Officer."; the use case ends.

4b. Alternative – Member is Reserved for a task and declares Available
1. The system keeps the status Reserved, updates only the location and displays the assigned task; the use case ends.

4c. Exception – Member is Reserved for a task and declares Not Available
1. The system rejects the change; the member stays Reserved and the task is unchanged.
2. The system displays "You are reserved for task <task> and cannot mark yourself unavailable. Contact the Logistics and Resources Officer." and records the rejected attempt in the audit log; the use case ends.

5a. Exception – The crew member was reserved, or recorded as unavailable, concurrently
1. The system finds on re-verification that the member is now Reserved for a task or has an Unavailable period; it discards the change and the flow resumes at step 4, so the request is checked against the current state.

---

## UC-11 View Assigned Task Details

### Overview
- ID: UC-11
- Primary Actor: Field Crew Member
- Supporting Actors: None
- Description: A Field Crew Member views the details of the task to which they are currently assigned.
- Trigger: The crew member selects "My Task".
- Relationships: None

### Details tab
Pre-conditions:
- The Field Crew Member is logged in.

Post-conditions (Success):
- The member sees the incident name, type and location, the task name, its state, the Field Commander, the other assigned resources and the estimated duration.

Post-conditions (Failure):
- No task details are shown.

### Flow of Events tab – Basic Flow
1. The Field Crew Member selects "My Task".
2. The system finds the task for which the member is Reserved.
3. The system displays the incident and task details.
4. The system pushes any change of the task or its resources to the screen while it is open.

### Extensions pane
2a. Alternative – Member is not assigned to any task
1. The system displays "You are not assigned to a task."; the use case ends.

---

## UC-12 Report Equipment Fault

### Overview
- ID: UC-12
- Primary Actors: Field Crew Member, Field Commander
- Supporting Actors: None
- Description: A member of the field team reports in real time that an equipment item broke down during an Active task. The item becomes Faulty/In Repair immediately and is released from the task, because a resource has exactly one status at any moment; the task stays Active and the Dispatcher is alerted to assign a replacement (D-10).
- Trigger: An equipment item breaks down during an Active task.
- Relationships: None. Usually followed by UC-28 Assign Replacement Resource performed by the Dispatcher.
- Requirements: FR-12.

### Details tab
Pre-conditions:
- The user is logged in as a Field Crew Member or Field Commander and is assigned to the Active task that uses the item.

Post-conditions (Success):
- The item's status is Faulty/In Repair and it can no longer be reserved.
- The item is no longer reserved for the task, its assignment record is closed with the fault time (the record still counts for costs and utilization), the package item it filled is missing, and the task stays Active with its other resources Reserved.
- The Dispatchers have received an alert that a replacement is required.
- The fault report is recorded in the audit log.

Post-conditions (Failure):
- The item's status and its reservation are unchanged and the reporter is told why.

### Flow of Events tab – Basic Flow
1. The user selects "Report Equipment Fault" on the assigned task.
2. The system displays the equipment items reserved for the task.
3. The user selects the faulty item and enters a short description.
4. The user submits the report.
5. The system finds the task for which the item is currently Reserved.
6. In one transaction, the system verifies that the item is still Reserved for that task and the task is Active.
7. The system sets the item's status to Faulty/In Repair, releases it from the task, closes its assignment record with the fault time and marks the package item it filled as missing; the task stays Active.
8. The system commits and records the report in the audit log.
9. The system pushes the change to all relevant workstations and sends the Dispatchers a "Replacement required" alert for the task.
10. The system confirms to the user that the fault was reported.

### Extensions pane
5a. Exception – Item is already Faulty/In Repair or no longer reserved for any task
1. The system displays "This item has already been reported as faulty or released."; the use case ends.

6a. Exception – The assignment changed between steps 5 and 6 (another operation came first)
1. The system rolls back; the flow resumes at step 5, so the report is checked against the current state.

6b. Exception – The task is no longer Active (completed or cancelled in the meantime)
1. The system rolls back and displays the task's current state; the use case ends.

---

## UC-28 Assign Replacement Resource

### Overview
- ID: UC-28
- Primary Actor: Dispatcher
- Supporting Actors: None
- Description: After a resource of an Active task is lost (an equipment item reported Faulty/In Repair, or a crew member, including the Field Commander, recorded as Unavailable), the Dispatcher fills the missing package item with an Available resource of the same equipment type or crew role, chosen manually or automatically, or confirms that the item is no longer needed (D-10).
- Trigger: The Dispatcher receives a "Replacement required" alert (result of UC-12 or UC-09).
- Relationships: Includes UC-25 Reserve Resources Atomically. Extended by UC-27 Retry with Next Best Alternative.
- Requirements: FR-03, FR-13, NFR-02, NFR-08.

### Details tab
Pre-conditions:
- The Dispatcher is logged in.
- The task is Active and at least one of its package items is missing.

Post-conditions (Success):
- Either an Available resource of the same equipment type or crew role is Reserved for the task, or the Dispatcher has confirmed that the item is no longer needed. The task stays Active and the alert is marked resolved.
- If the replacement fills the Field Commander item, the task's permissions have moved to the new Field Commander (view the task's details, report faults, report completion) and the previous Field Commander no longer has them.
- The change is recorded in the audit log and pushed to all relevant workstations.

Post-conditions (Failure):
- The item stays missing, the task stays Active and the alert stays open.

### Flow of Events tab – Basic Flow
1. The Dispatcher opens the "Replacement required" alert.
2. The system displays the task, the missing package item with the lost resource, and the Available resources of the same equipment type or crew role sorted by distance to the incident.
3. The Dispatcher selects a replacement resource and confirms.
4. The system reserves the resource for the missing item in one transaction (include UC-25 Reserve Resources Atomically, expected task state Active).
5. The system marks the item filled and the alert resolved, and records them in the audit log. If the item is the Field Commander, the system moves the task's permissions to the new Field Commander in the same transaction and removes them from the previous one.
6. The system pushes the change to all workstations displaying the incident.

### Extensions pane
2a. Exception – No Available resource of the required type or role
1. The system displays "No Available <type> in the pool."; the alert stays open.
2. The Dispatcher closes the alert screen (the use case ends), or confirms that the item is no longer needed (extension 3b).

3a. Alternative – Automatic selection
1. The Dispatcher selects "Auto-Select" and confirms the criterion ("Nearest to incident" is preselected).
2. The system selects the best-ranked Available resource for the missing item.
3. The system reserves it (include UC-25 Reserve Resources Atomically, expected task state Active).
4. If the resource was reserved concurrently or could not be reserved in time (extended by UC-27 Retry with Next Best Alternative), UC-27 is performed without any action by the Dispatcher. UC-27 retries at most 2 times, replacing only the resources that were taken and keeping the rest; if no alternative exists or the third attempt also loses to contention, nothing is reserved and the message distinguishes the two cases.
5. On success, the flow resumes at step 5.

3b. Alternative – The Dispatcher confirms that the item is no longer needed
1. The Dispatcher selects "No Longer Needed" and confirms.
2. In one transaction, the system verifies that the task is Active and the item is still missing.
3. The system marks the item as not needed, resolves the alert, commits and records it in the audit log.
4. The flow resumes at step 6.

3c. Exception – The missing item is the Field Commander and the Dispatcher selects "No Longer Needed"
1. A replacement Field Commander is mandatory: the system displays "A task must have a Field Commander. Assign a replacement."; the flow resumes at step 3.

4a. Exception – The selected resource was reserved concurrently or became unavailable
1. The system rolls back (UC-26 Roll Back Reservation), displays why the resource is unavailable, with the messages of UC-23 extension 6a, and refreshes the list; the flow resumes at step 3.

4b. Exception – The task is no longer Active, or the item is no longer missing (another Dispatcher already handled it)
1. The system rolls back and displays "This item has already been handled." with the task's current state, and closes the alert; the use case ends.

4c. Exception – The replacement cannot be completed within 5 seconds (D-03)
1. The system rolls back; nothing is reserved and the item stays missing.
2. The system displays "The reservation could not be completed in time; nothing was reserved. Please retry."; the flow resumes at step 3.
