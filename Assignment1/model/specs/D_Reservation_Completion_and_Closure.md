# CrisisSync – Use Case Specifications: Reservation, Completion and Closure

Use cases: UC-23 to UC-27 and UC-29 to UC-33.
Each specification follows the Visual Paradigm 18.1 *Use Case Details* layout:
Overview → Details tab (pre/post-conditions) → Flow of Events tab (basic flow) → Extensions pane (alternative and exception flows, each branching from a numbered step).

---

## UC-23 Reserve Resource Package

### Overview
- ID: UC-23
- Primary Actor: Dispatcher
- Supporting Actors: None
- Description: The Dispatcher manually selects specific resources for a Ready task's package, and the system reserves all of them and moves the task to Active in one all-or-nothing transaction.
- Trigger: The Dispatcher selects "Reserve Manually" on a Ready task in the incident status screen.
- Relationships: Includes UC-25 Reserve Resources Atomically.
- Requirements: FR-01, FR-02, FR-03, FR-04, FR-05, NFR-01, NFR-02, NFR-06, NFR-07, NFR-08.

### Details tab
Pre-conditions:
- The Dispatcher is logged in and viewing the incident status screen.
- The task is in state Ready (all its prerequisites are Completed).

Post-conditions (Success):
- In one committed transaction, every resource of the package is Reserved for this task and the task is Active.
- No resource is reserved for more than one task, and the task has exactly one reserved package.
- The change is recorded in the audit log and pushed to all workstations displaying the incident within 1 second.

Post-conditions (Failure):
- The transaction is rolled back: no resource of the package is reserved and the task keeps its current state.
- The Dispatcher sees the reason (unavailable resources, task no longer Ready, or timeout), and the screen shows the current state.

### Flow of Events tab – Basic Flow
1. The Dispatcher selects a Ready task and chooses "Reserve Manually".
2. The system displays the package requirements and, for each required role or equipment type, the resources that are currently Available with their current location.
3. The Dispatcher selects one resource for every item of the package, including exactly one Field Commander.
4. The Dispatcher confirms the reservation.
5. The system verifies that the user has the Dispatcher role and that the selection matches the package.
6. The system reserves the selected resources and sets the task to Active in one transaction (include UC-25 Reserve Resources Atomically).
7. The system pushes the new task and resource states to all workstations displaying the incident.
8. The system displays "Package reserved – task is Active" to the Dispatcher.

### Extensions pane
2a. Exception – Not enough Available resources of a required type
1. The system marks the package item that cannot be filled and disables confirmation.
2. The Dispatcher cancels the operation; the use case ends with the task still Ready.

4a. Alternative – Dispatcher switches to automatic allocation
1. The Dispatcher selects "Auto-Allocate" instead of confirming; UC-24 Auto-Allocate Resource Package starts and this use case ends.

5a. Exception – The request comes from a session without the Dispatcher role
1. The system rejects the request without reserving anything, displays "Only a Dispatcher can reserve resources." and records the rejected attempt in the audit log; the use case ends.

5b. Exception – Selection does not match the package
1. The system displays which item is missing or redundant (for example, "Exactly one Field Commander is required").
2. The flow resumes at step 3.

6a. Exception – A selected resource was taken concurrently or became unavailable (via UC-25, extended by UC-26 Roll Back Reservation)
1. The system rolls back the whole transaction; no resource is reserved and the task stays Ready.
2. The system displays one line per unavailable resource with its ID and name (a crew member's full name, or an equipment item's type): "<ID> <name> is already reserved for task <task>." for a resource Reserved for another task, "<ID> <name> is Faulty/In Repair." for a faulty equipment item, or "<ID> <name> is unavailable." for an Unavailable crew member.
3. The system refreshes the Dispatcher's screen to the current state.
4. The flow resumes at step 3 so the Dispatcher can pick replacements.

6b. Exception – The task is not Ready
1. Inside the transaction, the system finds that the task is not in state Ready: another Dispatcher has already reserved a package for it, it was cancelled, or it is still Blocked.
2. The system rolls back; no resource is reserved.
3. The system displays "Task <name> is no longer Ready." and refreshes the screen; the use case ends.

6c. Exception – The reservation cannot be completed within 5 seconds (D-03)
1. The system rolls back the transaction; nothing is reserved and the task stays Ready.
2. The system displays "The reservation could not be completed in time; nothing was reserved. Please retry."; the flow resumes at step 4.

---

## UC-24 Auto-Allocate Resource Package

### Overview
- ID: UC-24
- Primary Actor: Dispatcher
- Supporting Actors: None
- Description: The system chooses the resources for a Ready task's package according to a selected allocation criterion (default: nearest Available resource to the incident location), then reserves them and moves the task to Active in one transaction, within 5 seconds (D-03).
- Trigger: The Dispatcher selects "Auto-Allocate" on a Ready task.
- Relationships: Includes UC-25 Reserve Resources Atomically. Extended by UC-27 Retry with Next Best Alternative.
- Requirements: FR-02, FR-06, FR-07, NFR-03, NFR-11.

### Details tab
Pre-conditions:
- The Dispatcher is logged in and viewing the incident status screen.
- The task is in state Ready.
- At least one allocation criterion is configured (the default is geographic proximity).

Post-conditions (Success):
- In one committed transaction, every package item is Reserved for the task with the best-ranked Available resource and the task is Active.
- Changes are recorded in the audit log and pushed to all relevant workstations.

Post-conditions (Failure):
- No resource is reserved, the task keeps its current state and the Dispatcher is told why: no Available resource in the pool, resources lost to competing requests, or task no longer Ready.

### Flow of Events tab – Basic Flow
1. The Dispatcher selects a Ready task and chooses "Auto-Allocate".
2. The system displays the available allocation criteria with "Nearest to incident" preselected.
3. The Dispatcher confirms the criterion.
4. The system ranks the Available resources of each required role and equipment type by the selected criterion.
5. The system selects the best-ranked resource for every package item.
6. The system reserves the selected resources and sets the task to Active in one transaction (include UC-25 Reserve Resources Atomically).
7. The system records the criterion and the allocation result in the audit log.
8. The system pushes the change to all relevant workstations.
9. The system displays the allocated resources and their distances to the Dispatcher.

### Extensions pane
5a. Exception – No Available resource for a package item
1. The system displays "Auto-allocation failed: no Available <item> in the pool. Nothing was reserved."
2. The task remains Ready and the use case ends.

6a. Exception – A selected resource was reserved concurrently, or could not be reserved within the attempt's time limit (extended by UC-27 Retry with Next Best Alternative)
1. UC-27 is performed without any action by the Dispatcher; the flow resumes at step 7 if it succeeds. UC-27 retries at most 2 times, replacing only the resources that were taken and keeping the rest; if no alternative exists or the third attempt also loses to contention, nothing is reserved and the message distinguishes the two cases.

6b. Exception – The task is no longer Ready
1. The system rolls back, displays "Task <name> is no longer Ready." and refreshes the screen; the use case ends.

---

## UC-25 Reserve Resources Atomically (included)

### Overview
- ID: UC-25
- Primary Actor: None (included by UC-23, UC-24 and UC-28)
- Supporting Actors: None
- Description: In one all-or-nothing transaction the system re-checks the task, verifies that all requested resources are Available, reserves them and updates the task, then commits once. If any check fails, nothing changes.
- Trigger: A base use case submits a task ID, the expected task state (Ready for a new package, Active for a replacement of a missing item), a set of resource IDs and a time limit.
- Relationships: Included by UC-23, UC-24, UC-28. Extended by UC-26 Roll Back Reservation.
- Requirements: FR-02, FR-04, FR-10, FR-11, NFR-07, NFR-08.

### Details tab
Pre-conditions:
- The base use case supplies a task ID, the expected task state, a non-empty set of resource IDs (for a replacement: the missing package item to fill) and a time limit (D-03).

Post-conditions (Success):
- In a single commit: all requested resources are Reserved for the task with an open assignment record, the task is Active (if it was Ready) or the missing item is filled (replacement), and the audit entries are written.

Post-conditions (Failure):
- The transaction is rolled back; the incident, the task and every requested resource keep their previous state.

### Flow of Events tab – Basic Flow
1. The system starts a database transaction.
2. The system verifies that the task is still in the expected state and, for a replacement, that the package item is still missing.
3. The system verifies that every requested resource has status Available (a Field Commander already reserved for another task, a crew member marked unavailable, or a Faulty/In Repair item is not Available).
4. The system sets every requested resource to Reserved and opens an assignment record linking it to the task.
5. If the expected state is Ready, the system sets the task to Active; for a replacement, it marks the package item filled.
6. The system writes the audit entries for the reservation and the state transition.
7. The system commits all changes in one commit and returns "success" to the base use case.

### Extensions pane
2a. Exception – The task is not in the expected state, or the item is no longer missing (another reservation, replacement or cancellation committed first)
1. The system rolls back the transaction and returns "task changed" with the task's current state to the base use case.

3a. Exception – At least one resource is not Available
1. UC-26 Roll Back Reservation is performed.

3b. Exception – The reservation cannot be completed within the time limit (another operation is still working on the same task or resources)
1. The system rolls back the transaction and returns "timeout" with the IDs of the resources it could not reserve in time.

---

## UC-26 Roll Back Reservation (extension)

### Overview
- ID: UC-26
- Primary Actor: Dispatcher (through the extended use case UC-25, included by UC-23, UC-24 and UC-28)
- Supporting Actors: None
- Description: When any requested resource is not Available, the system cancels the whole reservation so that no resource of the package is reserved.
- Trigger: UC-25 finds an unavailable resource.
- Relationships: Extends UC-25 Reserve Resources Atomically. Condition: at least one requested resource is not Available when the reservation is made.

### Details tab
Pre-conditions:
- A reservation transaction is open and at least one requested resource is not Available.

Post-conditions (Success):
- The transaction is rolled back, and no resource status and no task state has changed.
- The base use case receives the list of unavailable resources.
- The failed attempt is recorded in the audit log.

Post-conditions (Failure):
- Not applicable.

### Flow of Events tab – Basic Flow
1. The system collects the IDs and names of all requested resources that are not Available.
2. The system rolls back the transaction.
3. The system records the failed reservation attempt in the audit log.
4. The system returns "unavailable" with the list of resources to the base use case.

### Extensions pane
None.

---

## UC-27 Retry with Next Best Alternative (extension)

### Overview
- ID: UC-27
- Primary Actor: Dispatcher (through the extended use cases UC-24 and UC-28)
- Supporting Actors: None
- Description: When another transaction takes a selected resource first, or it cannot be reserved in time, the system automatically replaces only the affected resources with the next-best Available alternatives, keeps the other selected resources, and retries the whole package as a new all-or-nothing transaction, at most two alternative attempts (three attempts in total), without blocking the server for other users (D-05).
- Trigger: An automatic attempt in UC-24 or UC-28 fails because a selected resource was reserved concurrently or could not be reserved in time.
- Relationships: Extends UC-24 Auto-Allocate Resource Package and UC-28 Assign Replacement Resource. Condition: the atomic reservation of an automatic selection returned "unavailable" or "timeout" (a timeout counts as losing the attempt to contention), and fewer than 2 retries have been made.
- Requirements: FR-07.

### Details tab
Pre-conditions:
- The previous automatic reservation attempt was rolled back because of a concurrent reservation or a timeout.
- Fewer than 2 retries have been performed for this request.

Post-conditions (Success):
- The resources are reserved with the next-best alternatives for the affected items and the kept resources for the others, and the base use case continues.

Post-conditions (Failure):
- When no alternative exists, or the third attempt also loses to contention, nothing is reserved, the task keeps its state (Ready in UC-24; Active with the item still missing in UC-28) and the Dispatcher sees the reason.

### Flow of Events tab – Basic Flow
1. The system removes only the resources that were taken or could not be reserved in time from the candidate ranking; every other selected resource is kept.
2. The system selects the next-best Available resource for each affected item.
3. The system increments the retry counter and records the retry in the audit log.
4. The system performs UC-25 Reserve Resources Atomically again for the whole package (the kept and the new resources) as a new all-or-nothing transaction.
5. The reservation succeeds; control returns to the base use case (UC-24 at step 7, or UC-28 at step 5).

### Extensions pane
2a. Exception – No alternative Available
1. The system reports "Auto-allocation failed: no Available <item> in the pool. Nothing was reserved."; the base use case ends.

5a. Exception – Conflict or timeout again (a kept resource that was taken in the meantime counts as affected)
1. If fewer than 2 retries were made, the flow resumes at step 1.
2. Otherwise (the third attempt also lost to contention) the system reports "Auto-allocation failed: resources were lost to competing requests. Nothing was reserved."; the base use case ends.

---

## UC-29 Report Task Completion

### Overview
- ID: UC-29
- Primary Actor: Field Commander
- Supporting Actors: None
- Description: The Field Commander assigned to an Active task reports that the task is finished; the system completes it, releases its resources and promotes dependent tasks. Completion is blocked while a package item is missing, unless the Dispatcher has confirmed that the item is no longer needed (D-10).
- Trigger: The Field Commander selects "Report Completion" on their task.
- Relationships: Includes UC-21 Release Task Resources; includes UC-30 Update Task Readiness.
- Requirements: FR-08, FR-09, FR-10, NFR-02, NFR-06, NFR-09, NFR-11.

### Details tab
Pre-conditions:
- The Field Commander is logged in and viewing the status screen of the incident (UC-18).

Post-conditions (Success):
- The task is Completed with a recorded completion timestamp.
- All resources still reserved for the task are Available and their assignment records are closed.
- Every dependent task that now has all its prerequisites Completed is Ready.
- All changes are recorded in the audit log and pushed to all relevant workstations within 1 second.

Post-conditions (Failure):
- The task stays Active and all its resources stay Reserved.

### Flow of Events tab – Basic Flow
1. The Field Commander selects "Report Completion" on their Active task.
2. The system asks for confirmation and shows the task name and its reserved resources.
3. The Field Commander confirms.
4. The system starts a transaction and verifies that the user is the task's assigned Field Commander, the task is Active and no package item is missing.
5. The system sets the task to Completed and records the completion timestamp.
6. The system releases the task's resources (include UC-21 Release Task Resources).
7. The system updates the readiness of the task's dependent tasks (include UC-30 Update Task Readiness).
8. The system commits steps 4–7 as one transaction and records them in the audit log.
9. The system pushes the changes to all workstations displaying the incident.
10. The system displays "Task completed" to the Field Commander.

### Extensions pane
3a. Alternative – Field Commander cancels
1. The system makes no change; the use case ends.

4a. Exception – User is not the assigned Field Commander
1. The system rolls back, rejects the request, displays "Only the assigned Field Commander can complete this task." and records the attempt in the audit log; the use case ends.

4b. Exception – Task is no longer Active
1. The system rolls back, displays "The task is already <state>." and refreshes the screen; the use case ends.

4c. Exception – A package item is missing
1. The system rolls back and displays "Task <name> cannot be completed: the <item> item has no resource. The Dispatcher must assign a replacement or confirm that it is no longer needed."; the use case ends.

---

## UC-30 Update Task Readiness (included)

### Overview
- ID: UC-30
- Primary Actor: None (included by UC-15 Modify Task Dependencies and UC-29 Report Task Completion)
- Supporting Actors: None
- Description: The system evaluates each candidate Blocked task and sets it to Ready if all its prerequisites are Completed; otherwise the task stays Blocked. The evaluation always runs; only its outcome depends on the prerequisites.
- Trigger: A task is completed, or a Blocked task's prerequisites change.
- Relationships: Included by UC-15, UC-29.
- Requirements: FR-09.

### Details tab
Pre-conditions:
- A set of candidate Blocked tasks is known (dependents of the completed task, or the edited task).

Post-conditions (Success):
- Every candidate task that has all its prerequisites Completed is Ready; the others remain Blocked.

Post-conditions (Failure):
- The base use case's transaction is rolled back.

### Flow of Events tab – Basic Flow
1. The system takes the next candidate task in state Blocked.
2. The system checks whether all its prerequisites are Completed.
3. The system sets the task to Ready and records the transition in the audit log.
4. The system repeats steps 1–3 for every candidate and returns to the base use case.

### Extensions pane
2a. Alternative – Not all prerequisites are Completed
1. The task stays Blocked; the flow continues with the next candidate at step 1.

---

## UC-31 Close Incident

### Overview
- ID: UC-31
- Primary Actor: Dispatcher
- Supporting Actors: National Clearing and Settlement System
- Description: The Dispatcher closes an incident whose tasks are all in a final state. Closing runs in three phases: a short transaction moves the incident to In Settlement (D-11), and only one Dispatcher can succeed in it; the settlement request is sent outside any transaction with the incident's unique ID, so a repeated request is never billed twice; a new transaction records the outcome (Closed on approval, back to Open on rejection or failure, including timeout).
- Trigger: The Dispatcher selects "Close Incident" on the incident status screen.
- Relationships: Includes UC-32 Calculate Operating Costs. Extended by UC-33 Handle Settlement Rejection.

### Details tab
Pre-conditions:
- The Dispatcher is logged in and viewing the incident status screen.

Post-conditions (Success):
- The settlement approval and its transaction token are stored in the database.
- The incident is Closed with an automatically recorded closing timestamp.
- The closing and settlement are recorded in the audit log.

Post-conditions (Failure):
- Closing already in progress (2a, 6a): the incident stays In Settlement under the other closing attempt; nothing is changed by this request and the Dispatcher sees "Closing is already in progress for this incident."
- Tasks not in a final state (2b): the incident stays Open, nothing is changed and the Dispatcher sees the reason.
- Dispatcher declines (5a): the incident stays Open, nothing is changed and no alert is sent.
- The incident changed before phase 1 (6b): it keeps the state the other operation gave it (Cancelled, or Open with a new non-final task), and the Dispatcher sees that state.
- Settlement rejected (8a, UC-33), or failed or timed out (8b): the incident is Open again without a closing timestamp, no settlement approval is stored, the outcome is recorded in the audit log and the Dispatcher receives an alert.

### Flow of Events tab – Basic Flow
1. The Dispatcher selects "Close Incident".
2. The system verifies that the incident is Open and every task of the incident is in a final state.
3. The system calculates the total operating costs (include UC-32 Calculate Operating Costs).
4. The system displays the cost summary and asks for confirmation.
5. The Dispatcher confirms.
6. Phase 1: in one short transaction, the system re-verifies that the incident is Open and all its tasks are final, stores the cost summary, sets the incident to In Settlement and commits. From now on the incident is not Open: it cannot be cancelled, closed again or given new tasks.
7. Phase 2: with no transaction open, the system sends a billing and budget-reimbursement request with the cost summary and the incident's unique ID to the National Clearing and Settlement System.
8. The National Clearing and Settlement System returns an approval with a transaction token.
9. Phase 3: in a new transaction, the system verifies that the incident is still In Settlement, stores the approval and the transaction token, sets the incident to Closed, records the closing timestamp automatically and commits.
10. The system records the closing and the settlement in the audit log, pushes the change and displays "Incident closed – settlement <token>".

### Extensions pane
2a. Exception – The incident is In Settlement
1. The system displays "Closing is already in progress for this incident."; the use case ends.

2b. Exception – Tasks not in a final state
1. The system displays "The incident cannot be closed: N tasks are not Completed or Cancelled." and lists them; the use case ends.

5a. Alternative – Dispatcher declines
1. The system makes no change; the use case ends.

6a. Exception – Another Dispatcher started closing the incident first
1. The system finds on re-verification that the incident is already In Settlement; only the first Dispatcher succeeds.
2. The system rolls back and displays "Closing is already in progress for this incident."; the use case ends.

6b. Exception – The incident changed in another way before phase 1 (it was cancelled, or a task was added)
1. The system rolls back, displays the incident's current state and the use case ends.

8a. Exception – The clearing system rejects the settlement (extended by UC-33 Handle Settlement Rejection)
1. UC-33 is performed; the use case ends with the incident Open again.

8b. Exception – The settlement fails: no response within 30 seconds (timeout) or a communication failure
1. Phase 3: in a new transaction, the system sets the incident back to Open and commits.
2. The system records the failure in the audit log and alerts the Dispatcher "Settlement could not be completed – the incident is open again. Please retry."; the use case ends.
3. A response that arrives later is recorded in the audit log and changes nothing. If it was an approval, the retry carries the same incident ID, so the clearing system returns that approval and token without billing again.

---

## UC-32 Calculate Operating Costs (included)

### Overview
- ID: UC-32
- Primary Actor: None (included by UC-31 Close Incident)
- Supporting Actors: None
- Description: The system computes the total operating cost of an incident from personnel hours, engine hours of engineering and medical equipment, and consumables.
- Trigger: UC-31 requests the cost summary.
- Relationships: Included by UC-31.

### Details tab
Pre-conditions:
- All tasks of the incident are in a final state and all their assignment records are closed.

Post-conditions (Success):
- A cost summary with the three cost components and the total is returned to UC-31.

Post-conditions (Failure):
- Not applicable.

### Flow of Events tab – Basic Flow
1. The system sums, for every crew member, the hours of the member's assignment records for tasks of the incident (from reservation to release), multiplied by the role's hourly rate.
2. The system sums, for every equipment item, the engine hours recorded on its assignment records for tasks of the incident, multiplied by the equipment type's hourly rate. Records closed early because a resource was released during a task (D-10) are included in steps 1 and 2.
3. The system sums the consumables recorded for the incident.
4. The system returns the three components and the total to UC-31.

### Extensions pane
None.

---

## UC-33 Handle Settlement Rejection (extension)

### Overview
- ID: UC-33
- Primary Actor: Dispatcher (through the extended use case UC-31; receives the alert)
- Supporting Actors: National Clearing and Settlement System (through the extended use case UC-31)
- Description: When the clearing system rejects the settlement, the incident returns from In Settlement to Open and the Dispatcher is alerted with the rejection reason.
- Trigger: The clearing system returns a rejection.
- Relationships: Extends UC-31 Close Incident. Condition: the settlement response is a rejection.

### Details tab
Pre-conditions:
- UC-31 sent a settlement request, the incident is In Settlement and the clearing system returned a rejection.

Post-conditions (Success):
- The incident is Open again without a closing timestamp, the rejection is recorded in the audit log and the Dispatcher sees the reason.

Post-conditions (Failure):
- Not applicable.

### Flow of Events tab – Basic Flow
1. Phase 3: in a new transaction, the system verifies that the incident is still In Settlement.
2. The system sets the incident back to Open without a closing timestamp, records the rejection and its reason in the audit log and commits.
3. The system alerts the Dispatcher: "Settlement rejected: <reason>. The incident remains open."
4. The use case ends; the Dispatcher may retry UC-31 later. The retry carries the same incident ID; nothing was billed for the rejected request.

### Extensions pane
None.
