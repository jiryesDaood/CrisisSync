# CrisisSync – Use Case Specifications: Incidents and Task Trees

Use cases: UC-13 to UC-22.
Each specification follows the Visual Paradigm 18.1 *Use Case Details* layout:
Overview → Details tab (pre/post-conditions) → Flow of Events tab (basic flow) → Extensions pane (alternative and exception flows, each branching from a numbered step).

---

## UC-13 Open Incident

### Overview
- ID: UC-13
- Primary Actor: Dispatcher
- Supporting Actors: None
- Description: The Dispatcher opens a new emergency incident so that a task tree can be built and resources can be reserved for it.
- Trigger: The Dispatcher receives a report of an emergency event and selects "Open Incident".
- Relationships: None

### Details tab
Pre-conditions:
- The Dispatcher is logged in (UC-01).

Post-conditions (Success):
- A new incident exists with status Open, a unique ID, name, type, location and an automatically recorded opening timestamp.
- A live status screen for the incident is available to every connected Dispatcher workstation.
- The opening is recorded in the audit log with timestamp and user.

Post-conditions (Failure):
- No incident is created and no audit entry for an incident opening is written.

### Flow of Events tab – Basic Flow
1. The Dispatcher selects "Open Incident".
2. The system displays the incident form with the closed list of incident types: Building Collapse, Fire, Road Accident, Hazmat Leak, Natural Disaster.
3. The Dispatcher enters the incident name, selects the incident type and enters the location.
4. The Dispatcher submits the form.
5. The system validates that all mandatory fields are filled and the location is valid.
6. The system creates the incident with status Open and records the opening timestamp automatically.
7. The system writes an audit entry and pushes the new incident status screen to all connected Dispatcher workstations.
8. The system displays the incident status screen with an empty task tree.

### Extensions pane
4a. Alternative – Dispatcher abandons the form
1. The Dispatcher selects "Cancel" before submitting.
2. The system discards the form; the use case ends without creating an incident.

5a. Exception – Missing or invalid data
1. The system detects that a mandatory field is empty or the location cannot be resolved.
2. The system highlights the invalid fields and displays an explanatory message.
3. The Dispatcher corrects the data; the flow resumes at step 4.

---

## UC-14 Create Task

### Overview
- ID: UC-14
- Primary Actor: Dispatcher
- Supporting Actors: None
- Description: The Dispatcher adds a task to an open incident's task tree, defining its estimated duration, prerequisite tasks and required resource package.
- Trigger: The Dispatcher selects "Add Task" on the incident status screen.
- Relationships: Includes UC-16 Validate Dependency Graph.

### Details tab
Pre-conditions:
- The Dispatcher is logged in.
- The incident is in status Open.

Post-conditions (Success):
- The task exists in the incident's task tree with name, estimated duration, prerequisites and a resource package that contains exactly one Field Commander.
- The task's initial state is Ready if all its prerequisites are Completed (or it has none), otherwise Blocked.
- The change is pushed to all workstations displaying the incident and recorded in the audit log.

Post-conditions (Failure):
- The task tree is unchanged.

### Flow of Events tab – Basic Flow
1. The Dispatcher selects "Add Task" on the incident status screen.
2. The system displays the task form, the list of existing tasks of the incident, the closed list of crew roles (Paramedic, Heavy Equipment Operator, Firefighter, Hazmat Specialist, Rescuer, Field Commander) and the closed list of equipment types (Ambulance, Fire Truck, Bulldozer, Decontamination Tent, Generator).
3. The Dispatcher enters the task name and estimated duration.
4. The Dispatcher selects zero or more prerequisite tasks.
5. The Dispatcher defines the required resource package: one Field Commander plus the required quantities of each crew role and equipment type.
6. The Dispatcher submits the task.
7. The system validates that the package contains exactly one Field Commander and that all fields are valid.
8. The system starts a transaction and verifies that the incident is still Open.
9. The system validates the dependency graph (include UC-16 Validate Dependency Graph).
10. The system saves the task and sets its state to Ready if all its prerequisites are Completed (or it has none), otherwise to Blocked.
11. The system commits, records the state transition in the audit log and pushes the updated task tree to all workstations displaying the incident.

### Extensions pane
7a. Exception – Package does not contain exactly one Field Commander
1. The system displays "A resource package must contain exactly one Field Commander."
2. The Dispatcher corrects the package; the flow resumes at step 6.

7b. Exception – Invalid task data
1. The system highlights the invalid field (for example, a non-positive estimated duration).
2. The Dispatcher corrects the data; the flow resumes at step 6.

8a. Exception – The incident is no longer Open (closing is in progress, or it was closed or cancelled)
1. The system rolls back and displays "Tasks cannot be added: the incident is <status>."; the use case ends.

9a. Exception – The dependency would create a cycle (handled in UC-16, extended by UC-17 Reject Cyclic Dependency)
1. The system rolls back, keeps the task tree unchanged and displays the cycle.
2. The Dispatcher changes the prerequisites; the flow resumes at step 6.

9b. Exception – A selected prerequisite is Cancelled (reported by UC-16)
1. The system rolls back, keeps the task tree unchanged and displays "Task <name> is Cancelled and cannot be a prerequisite."
2. The Dispatcher changes the prerequisites; the flow resumes at step 6.

---

## UC-15 Modify Task Dependencies

### Overview
- ID: UC-15
- Primary Actor: Dispatcher
- Supporting Actors: None
- Description: The Dispatcher adds or removes prerequisite tasks of a Blocked task.
- Trigger: The Dispatcher selects "Edit Prerequisites" on a task in the incident status screen.
- Relationships: Includes UC-16 Validate Dependency Graph; includes UC-30 Update Task Readiness.

### Details tab
Pre-conditions:
- The Dispatcher is logged in.
- The incident is Open.

Post-conditions (Success):
- The task's prerequisites are updated and the dependency graph remains acyclic.
- If all its prerequisites are Completed, the task is in state Ready.
- The change is pushed to all relevant workstations and recorded in the audit log.

Post-conditions (Failure):
- The task's prerequisites and state are unchanged.

### Flow of Events tab – Basic Flow
1. The Dispatcher selects a task and chooses "Edit Prerequisites".
2. The system verifies that the task is in state Blocked.
3. The system displays the task's current prerequisites and the other tasks of the incident.
4. The Dispatcher adds and/or removes prerequisite tasks.
5. The Dispatcher confirms the change.
6. The system starts a transaction and re-verifies that the task is still Blocked.
7. The system validates the dependency graph (include UC-16 Validate Dependency Graph).
8. The system saves the new prerequisites.
9. The system re-evaluates the task's readiness (include UC-30 Update Task Readiness).
10. The system commits, records the change in the audit log and pushes the updated task tree to all workstations displaying the incident.

### Extensions pane
2a. Exception – Dependencies are locked
1. The system detects that the task is Ready, Active, Completed or Cancelled.
2. The system displays "Prerequisites can be changed only for a Blocked task."; the use case ends.

6a. Exception – Concurrent change of the task state
1. The system finds on re-verification that another user changed the task state since step 2 (for example, the task was cancelled).
2. The system rolls back, informs the Dispatcher and refreshes the screen to the current state; the use case ends.

7a. Exception – The change would create a cycle (handled in UC-16, extended by UC-17 Reject Cyclic Dependency)
1. The system rolls back, keeps the previous prerequisites and displays the cycle.
2. The Dispatcher revises the selection; the flow resumes at step 5.

7b. Exception – A selected prerequisite is Cancelled (reported by UC-16)
1. The system rolls back, keeps the previous prerequisites and displays "Task <name> is Cancelled and cannot be a prerequisite."
2. The Dispatcher revises the selection; the flow resumes at step 5.

---

## UC-16 Validate Dependency Graph (included)

### Overview
- ID: UC-16
- Primary Actor: None (included by UC-14 Create Task and UC-15 Modify Task Dependencies)
- Supporting Actors: None
- Description: The system checks that the incident's task dependencies, including the requested change, still form a directed acyclic graph (DAG).
- Trigger: A base use case submits a new or changed set of prerequisites.
- Relationships: Included by UC-14, UC-15. Extended by UC-17 Reject Cyclic Dependency.

### Details tab
Pre-conditions:
- A proposed set of prerequisite links for one task is available.

Post-conditions (Success):
- The proposed links are confirmed acyclic and may be saved by the base use case.

Post-conditions (Failure):
- A cycle is reported to the base use case; nothing is saved.

### Flow of Events tab – Basic Flow
1. The system builds the incident's dependency graph including the proposed links.
2. The system checks that every prerequisite belongs to the same incident and is not Cancelled.
3. The system searches the graph for a cycle.
4. The system finds no cycle and returns "valid" to the base use case.

### Extensions pane
2a. Exception – Prerequisite from another incident
1. The system returns "invalid: prerequisite belongs to another incident" to the base use case.

2b. Exception – A prerequisite is Cancelled
1. The system returns "invalid: prerequisite <name> is Cancelled" to the base use case.

3a. Exception – Cycle found
1. UC-17 Reject Cyclic Dependency is performed.

---

## UC-17 Reject Cyclic Dependency (extension)

### Overview
- ID: UC-17
- Primary Actor: Dispatcher (through the extended use case UC-16, included by UC-14 and UC-15)
- Supporting Actors: None
- Description: When a requested dependency creates a cycle, the system rejects the operation and tells the Dispatcher which tasks form the cycle.
- Trigger: UC-16 detects a cycle.
- Relationships: Extends UC-16 Validate Dependency Graph. Condition: the proposed graph contains a cycle.

### Details tab
Pre-conditions:
- UC-16 has detected at least one cycle in the proposed dependency graph.

Post-conditions (Success):
- The operation is rejected, the task tree is unchanged and the Dispatcher sees the tasks that form the cycle.

Post-conditions (Failure):
- Not applicable (the rejection itself is the outcome).

### Flow of Events tab – Basic Flow
1. The system discards the proposed prerequisite links.
2. The system displays "This dependency creates a cycle:" followed by the ordered list of task names in the cycle.
3. The system returns control to the base use case at the step where the Dispatcher edits the prerequisites.

### Extensions pane
None.

---

## UC-18 Monitor Incident Status

### Overview
- ID: UC-18
- Primary Actors: Dispatcher, Field Commander (restricted to the tasks they command)
- Supporting Actors: None
- Description: The user watches a live status screen of an open incident that the server updates by push whenever a task or resource changes. A Field Commander sees full details of, and can act on, only the tasks they command; when a Field Commander is replaced (UC-28), this scope moves to the new commander immediately.
- Trigger: The user opens an incident from the list of incidents available to them.
- Relationships: None. The Dispatcher starts UC-19, UC-22, UC-23, UC-24, UC-28 and UC-31 from this screen; the Field Commander starts UC-29.

### Details tab
Pre-conditions:
- The user is logged in as a Dispatcher or a Field Commander.

Post-conditions (Success):
- The user sees the current incident state within their permissions, and the screen keeps updating through server push without client polling.
- A Field Commander sees resources, elapsed time and actions only for tasks they command.

Post-conditions (Failure):
- The status screen is not shown; the user remains on the incident list.

### Flow of Events tab – Basic Flow
1. The user requests the list of incidents.
2. The system displays all Open and In Settlement incidents for a Dispatcher, or only the incidents in which the user commands a task for a Field Commander.
3. The user selects an incident.
4. The system displays the incident name, type, location, status (Open or In Settlement) and time elapsed since opening.
5. The system displays the progress indicator: number of Completed tasks out of all tasks.
6. For a Dispatcher, the system displays the task tree with its dependencies and, for each task, its state (with a distinct colour per state), Field Commander, reserved resources, missing package items (highlighted), and elapsed time against estimated duration.
7. The system highlights every displayed Active task whose elapsed time exceeds its estimated duration.
8. The system pushes every subsequent task or resource change to the screen within 1 second, and the screen updates without user action.
9. The user closes the status screen; the system stops pushing updates to this workstation.

### Extensions pane
2a. Alternative – No open incidents
1. The system displays "There are no open incidents."; the use case ends.

3a. Exception – Field Commander selects an incident outside their scope
1. The system denies access, displays "You are not assigned to a task in this incident." and records the attempt in the audit log; the flow resumes at step 2.

6a. Alternative – The user is a Field Commander
1. The system displays the task tree with its dependencies, showing only the name and state of each task.
2. For the tasks the user commands, the system also displays the reserved resources, missing package items and elapsed time against estimated duration.
3. The system offers task actions (Report Completion, Report Equipment Fault) only on the tasks the user commands; the flow continues at step 7.

---

## UC-19 Cancel Task

### Overview
- ID: UC-19
- Primary Actor: Dispatcher
- Supporting Actors: None
- Description: The Dispatcher cancels a task that is not in a final state and the system releases its resources. The dependent tasks shown before confirmation are only a preview; the system recomputes them on confirmation and aborts if they changed.
- Trigger: The Dispatcher selects "Cancel Task" on a task in the incident status screen.
- Relationships: Includes UC-21 Release Task Resources. Extended by UC-20 Cancel Dependent Tasks.
- Requirements: FR-11.

### Details tab
Pre-conditions:
- The Dispatcher is logged in and viewing the incident status screen (UC-18).

Post-conditions (Success):
- The task is Cancelled and every resource that was reserved for it is Available.
- Every non-final task that depends on it, directly or indirectly, is Cancelled and its resources are Available; these are exactly the dependent tasks the Dispatcher confirmed.
- All transitions are recorded in the audit log and pushed to all relevant workstations.

Post-conditions (Failure):
- No task state and no resource state is changed.

### Flow of Events tab – Basic Flow
1. The Dispatcher selects a task and chooses "Cancel Task".
2. The system verifies that the task is not in a final state.
3. The system determines all non-final tasks that depend on the task directly or indirectly (preview).
4. The system asks the Dispatcher to confirm the cancellation of the task.
5. The Dispatcher confirms.
6. The system starts a transaction, re-verifies that the task is not in a final state and recomputes its non-final dependent tasks.
7. The system sets the task to Cancelled (a reservation of the same task that has not committed yet is then rejected by UC-25).
8. The system releases the task's resources (include UC-21 Release Task Resources).
9. The system commits, records the transitions in the audit log and pushes the change to all relevant workstations.

### Extensions pane
2a. Exception – Task already in a final state
1. The system displays "The task is already Completed or Cancelled."; the use case ends.

3a. Alternative – Dependent tasks exist (extended by UC-20 Cancel Dependent Tasks)
1. UC-20 Cancel Dependent Tasks is performed; if it succeeds, the flow resumes at step 7 in the transaction UC-20 opened.

5a. Alternative – Dispatcher declines
1. The Dispatcher selects "No".
2. The system makes no change; the use case ends.

6a. Exception – The task reached a final state in the meantime
1. The system rolls back and displays "The task is already Completed or Cancelled."; the use case ends.

6b. Exception – Dependent tasks now exist although the preview showed none
1. The system rolls back without changing anything and displays "Task <name> now has M dependent tasks. Nothing was cancelled."
2. The flow resumes at step 3 with the new preview.

---

## UC-20 Cancel Dependent Tasks (extension)

### Overview
- ID: UC-20
- Primary Actor: Dispatcher (through the extended use case UC-19)
- Supporting Actors: None
- Description: When the cancelled task has non-final dependent tasks, the system tells the Dispatcher how many tasks will be cancelled (a preview) and, after confirmation, recomputes them and cancels them as well. If they changed since the preview, nothing is cancelled and the Dispatcher must confirm again.
- Trigger: UC-19 finds at least one non-final dependent task.
- Relationships: Extends UC-19 Cancel Task. Condition: the task has one or more dependent tasks that are not in a final state. Includes UC-21 Release Task Resources.

### Details tab
Pre-conditions:
- UC-19 has identified, as a preview, N ≥ 1 non-final tasks that depend on the task directly or indirectly.

Post-conditions (Success):
- After confirmation, the N dependent tasks are Cancelled and their resources are Available, in the same transaction as the original task.

Post-conditions (Failure):
- If the Dispatcher declines, or the dependent tasks changed since the preview, no task is cancelled (including the original task).

### Flow of Events tab – Basic Flow
1. The system displays "Cancelling this task will also cancel N dependent tasks:" followed by their names.
2. The Dispatcher confirms.
3. The system starts a transaction, re-verifies that the task is not in a final state, and recomputes its non-final dependent tasks.
4. The system verifies that the recomputed dependent tasks are exactly the N tasks shown in step 1.
5. The system sets each dependent task to Cancelled.
6. The system releases the resources of each dependent task (include UC-21 Release Task Resources).
7. The system returns to UC-19 at step 7 in the same transaction.

### Extensions pane
2a. Alternative – Dispatcher declines
1. The system cancels nothing and ends UC-19.

3a. Exception – The task reached a final state in the meantime
1. The system rolls back, displays "The task is already Completed or Cancelled." and ends UC-19.

4a. Exception – The dependent tasks changed since the preview
1. The system rolls back without changing anything and displays "The number of dependent tasks changed from N to M. Nothing was cancelled."
2. If M ≥ 1, the flow resumes at step 1 with the new preview; if M = 0, UC-19 resumes at step 4.

---

## UC-21 Release Task Resources (included)

### Overview
- ID: UC-21
- Primary Actor: None (included by UC-19 Cancel Task, UC-20 Cancel Dependent Tasks, UC-22 Cancel Incident and UC-29 Report Task Completion)
- Supporting Actors: None
- Description: The system returns every resource still reserved for a task to status Available and closes its assignment record.
- Trigger: A base use case moves a task to a final state.
- Relationships: Included by UC-19, UC-20, UC-22, UC-29.
- Requirements: FR-09.

### Details tab
Pre-conditions:
- The task has just been moved to Completed or Cancelled in the same transaction of the base use case.

Post-conditions (Success):
- Every crew member and equipment item still reserved for the task is Available, and its assignment record is closed with the release time.
- Resources already released during the task (lost resources, D-10) are not affected.

Post-conditions (Failure):
- The whole transaction of the base use case is rolled back.

### Flow of Events tab – Basic Flow
1. The system retrieves all resources still reserved for the task.
2. For each crew member, the system sets the status to Available.
3. For each equipment item, the system sets the status to Available.
4. The system closes the assignment record of each resource with the release time and records one audit entry per resource.
5. The system commits together with the base use case and pushes the resource changes.

### Extensions pane
1a. Alternative – No resources reserved (task was Blocked or Ready, or every resource was already released)
1. The system skips steps 2–4; control returns to the base use case.

---

## UC-22 Cancel Incident

### Overview
- ID: UC-22
- Primary Actor: Dispatcher
- Supporting Actors: None
- Description: The Dispatcher cancels an open incident at any stage (for example, a false alarm); all non-final tasks are cancelled and their resources released. An incident that is In Settlement cannot be cancelled (D-11).
- Trigger: The Dispatcher selects "Cancel Incident" on the incident status screen.
- Relationships: Includes UC-21 Release Task Resources.

### Details tab
Pre-conditions:
- The Dispatcher is logged in and viewing the incident status screen.

Post-conditions (Success):
- The incident is Cancelled with an automatically recorded cancellation timestamp.
- Every task that was not in a final state is Cancelled and its resources are Available.
- The cancellation is recorded in the audit log and pushed to all relevant workstations.

Post-conditions (Failure):
- The incident, its tasks and resources are unchanged.

### Flow of Events tab – Basic Flow
1. The Dispatcher selects "Cancel Incident".
2. The system verifies that the incident is Open, displays the number of tasks that are not in a final state and asks for confirmation.
3. The Dispatcher confirms.
4. The system starts a transaction and re-verifies that the incident is Open.
5. The system sets every non-final task of the incident to Cancelled.
6. The system releases the resources of each cancelled task (include UC-21 Release Task Resources).
7. The system sets the incident to Cancelled and records the cancellation timestamp automatically.
8. The system commits, records all transitions in the audit log and pushes the change to all relevant workstations.

### Extensions pane
2a. Exception – The incident is In Settlement
1. The system displays "The incident is being settled and cannot be cancelled."; the use case ends.

3a. Alternative – Dispatcher declines
1. The system makes no change; the use case ends.

4a. Exception – The incident is no longer Open (closing started or it was cancelled in the meantime)
1. The system rolls back and displays the incident's current status; the use case ends.
