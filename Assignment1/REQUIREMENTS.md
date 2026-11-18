# CrisisSync – Requirements, Traceability Matrix and Acceptance Tests

**Course:** Software Engineering Methods – Assignment 1 (Part B)
**System:** CrisisSync – Distributed Emergency Response Management
**Group:** G14

### Team Members

| # | Full Name | ID Number | College Email |
| :-: | :--- | :--- | :--- |
| 1 | Jiryes Daood | 213559677 | Jiryes.Daood@e.braude.ac.il |
| 2 | Nati Tehaok | 207785825 | nati.tehaok@e.braude.ac.il |
| 3 | Adam Zayyad | 325488559 | Adam.zayyad@e.braude.ac.il |
| 4 | mohamad dabdoub | 214030090 | mohamad.dabdoub@e.braude.ac.il |

---

## 1. Scope

This document defines requirements and acceptance tests for two CrisisSync operations:

| Operation | Description | Realising use cases (Visual Paradigm model) |
| :--- | :--- | :--- |
| **OP-1** | Reserve a resource package for a task, including reporting task completion | UC-23 Reserve Resource Package, UC-24 Auto-Allocate Resource Package, UC-25 Reserve Resources Atomically, UC-26 Roll Back Reservation, UC-27 Retry with Next Best Alternative, UC-28 Assign Replacement Resource, UC-29 Report Task Completion, UC-21 Release Task Resources, UC-30 Update Task Readiness, UC-01 Log In, UC-09 Update Crew Member Availability, UC-12 Report Equipment Fault, UC-19 Cancel Task |
| **OP-2** | Produce the SLA (on-time performance and service level) report | UC-34 Generate Monthly Reports, UC-35 Generate SLA Compliance Report, UC-06 View Management Reports |

Requirement keywords follow RFC 2119: **shall** marks a mandatory requirement.

---

## 2. Definitions and Design Decisions

The project story leaves the following points open. The decisions below make the requirements testable. Where the lecturer answered a clarification question on the course forum, the decision follows that answer, which overrides the story. These rows are marked *Lecturer clarification* with the number of her answer (forum thread "מטלה 1, שאלות הבהרה", 15.11.2026): D-02, D-05, D-07, D-08, D-10, D-11.

| ID | Topic | Decision |
| :--- | :--- | :--- |
| **D-01** | Available resource | A resource is *Available* only if its status is Available. A crew member who is Reserved or Unavailable (leave, sick, not ready), and an equipment item that is Reserved or Faulty/In Repair, is not Available. A Field Commander who is Reserved for an Active task is not Available for any other task. |
| **D-02** | Lock order and atomic reservation | *Lecturer clarification (answers 2, 6).* Every operation that changes a task's reservation locks the incident record (`SELECT ... FOR UPDATE`): reserving resources (including a replacement for a missing package item), reporting completion, cancelling a task, and releasing a lost resource. To prevent deadlock, all these operations lock in the same order: the incident record, then the affected task records, then the affected resource records in ascending ID order. Operations on the same incident are therefore applied one after the other: the first one is executed and the second is checked against the updated state (for example, a reservation and a cancellation of the same task). An operation that starts from a resource (a field fault report, an availability update by the Logistics and Resources Officer) first looks up, without a lock, which task the resource is reserved for, then locks in this order and re-verifies that the assignment has not changed. A package is reserved in one SQL transaction: lock the incident, lock the task and re-verify that it is Ready, lock all requested resources, check that all are Available, set them to Reserved and set the task to Active, commit once. Any failure rolls back everything. |
| **D-03** | Lock wait limits | Manual reservation (UC-23) and manual replacement (UC-28) wait at most **5 seconds** for all their row locks (incident, task and resources), then roll back and inform the Dispatcher. Automatic allocation (UC-24, and UC-28 with "Auto-Select") makes at most 3 attempts, and each attempt waits for its locks at most **1.5 seconds**; a lock timeout counts as losing that attempt to contention (D-05). There is no separate deadline: the lock waits add up to at most 4.5 s, so the allocation finishes within the **5 seconds** of NFR-03. |
| **D-04** | Auto-allocation criterion | The default criterion is the shortest straight-line distance between the resource's current location and the incident location. Criteria are pluggable so that new ones can be added later. |
| **D-05** | Auto-allocation conflict | *Lecturer clarification (answer 3).* When a concurrent transaction takes a selected resource, or its lock is not obtained within the attempt's 1.5-second lock wait (both count as losing the attempt to contention), the request retries automatically without blocking other clients' requests. Each alternative attempt replaces only the resources that were taken and keeps the rest of the selection, and is a new all-or-nothing transaction on the whole package. At most **2 alternative attempts** are made (3 attempts in total). Allocation fails in exactly two cases, each with its own message: a package item has no Available alternative in the pool ("Auto-allocation failed: no Available <item> in the pool. Nothing was reserved."), or the third attempt also loses to contention ("Auto-allocation failed: resources were lost to competing requests. Nothing was reserved."). There is no separate deadline abort; the time limits are those of D-03. |
| **D-06** | Completion idempotency | A completion report for a task that is already Completed has no effect and returns the current state. |
| **D-07** | SLA measure | *Lecturer clarification (answer 4).* For each incident: *time to last task completion* = (latest Completed-task timestamp) − (incident opening timestamp). A task cancelled after the last completion does not extend the interval. |
| **D-08** | SLA qualifying incident | *Lecturer clarification (answer 4).* An incident qualifies for month M if it was finally closed in M: its status is Closed (settlement approved, D-11), its closing timestamp falls within M (server time), and at least one of its tasks is Completed. Cancelled incidents and incidents that are Open or In Settlement do not qualify. The measured interval is still the one in D-07: from the incident opening to its last Completed task, not to the closing time; cancelled tasks are not taken into account. |
| **D-09** | Monthly schedule | Automatic monthly reports are generated at **00:05 server time on the 1st day** of each month for the previous calendar month. Missed runs are caught up when the server starts. |
| **D-10** | Resource lost during an Active task | *Lecturer clarification (answers 1, 5).* The full package must be reserved for a task to become Active. A task's package consists of package items, one per required crew role or equipment unit. When an equipment item reserved for an Active task is reported Faulty/In Repair by the field team (UC-12), or the Logistics and Resources Officer records a crew member reserved for it, including its Field Commander, as Unavailable (UC-09), the task stays Active. Because a resource has exactly one status at any moment, the resource is released from the task immediately: in one transaction the system sets its new status, closes its assignment record with an end time, and marks the package item as *missing*. Closed assignment records still count for the operating costs (UC-32) and the utilization report (UC-36). All Dispatchers receive a "Replacement required" alert. A Dispatcher fills the missing item with an Available resource of the same role or equipment type, chosen manually or automatically (UC-28), or confirms that the item is no longer needed to complete the task. The task cannot be completed while an item is missing. If the lost resource is the Field Commander, a replacement Field Commander is mandatory, so this item cannot be confirmed as no longer needed, and the task's permissions move to the new Field Commander. |
| **D-11** | In Settlement and Closed | *Lecturer clarification (answer 2).* *In Settlement*: closing of the incident (UC-31) is in progress; an incident In Settlement is not Open. *Closed*: the settlement was approved and the closing timestamp is recorded. |

---

## 3. Functional Requirements (FR)

### 3.1 OP-1 – Reserve a resource package and report task completion

| ID | Requirement |
| :--- | :--- |
| **FR-01** | The system shall accept a resource package reservation request only from a user with the Dispatcher role and only for a task in state Ready. |
| **FR-02** | The system shall reserve a task's package in one all-or-nothing SQL transaction that locks the records it changes in the lock order of D-02, re-verifies that the task is Ready, verifies that every requested resource is Available, sets each of them to Reserved for that task and sets the task to Active, committing all of these changes together. |
| **FR-03** | When a manual reservation or a manual replacement finds at least one selected resource not Available when locked, the system shall roll back the entire transaction so that none of the selected resources is reserved, keep the task in its previous state (Ready, or Active with the item still missing), display to the Dispatcher the ID and name of each unavailable resource, and refresh the Dispatcher's status screen to the current state. Automatic allocation reports its failures as defined in FR-07. |
| **FR-04** | The system shall reject a reservation that includes a Field Commander who is already Reserved for another task, a crew member who is Unavailable, or an equipment item that is Faulty/In Repair. |
| **FR-05** | After a reservation transaction commits, the system shall push the new task state (Active) and the new resource states to every connected workstation that displays the incident. |
| **FR-06** | When the Dispatcher requests automatic allocation, the system shall select for each package item the Available resource ranked best by the selected criterion (default: shortest distance to the incident location) and reserve the selection as defined in FR-02. |
| **FR-07** | If an auto-selected resource is reserved by a concurrent transaction or cannot be locked within the attempt's lock wait of 1.5 seconds (both count as losing the attempt to contention), the system shall automatically retry without a new Dispatcher request, at most 2 times per request, replacing only the affected resources with the next-best Available alternatives and keeping the other selected resources. Each attempt shall be a new all-or-nothing transaction on the whole package. The system shall report failure in exactly two cases, with distinct messages: "Auto-allocation failed: no Available <item> in the pool. Nothing was reserved." when a package item has no Available alternative, and "Auto-allocation failed: resources were lost to competing requests. Nothing was reserved." when the third attempt also loses to contention. |
| **FR-08** | The system shall accept a completion report for a task only from the Field Commander assigned to that task, only while the task is in state Active, and only if none of its package items is missing (D-10). |
| **FR-09** | On an accepted completion report, the system shall, in one transaction, set the task to Completed with the completion timestamp, set every resource still reserved for the task to Available and close its assignment record, and set to Ready every dependent task that has all its prerequisites Completed. |
| **FR-10** | The system shall record an audit-log entry with timestamp and user ID for every reservation, failed reservation, auto-allocation retry and task completion. |
| **FR-11** | The system shall reserve at most one resource package per task: when several reservation requests for the same task, or a reservation and a cancellation of the same task, arrive concurrently, the system shall apply them one at a time in the order they lock the incident record, and shall reject a reservation that finds the task no longer Ready without reserving any resource. |
| **FR-12** | When an equipment item reserved for an Active task is reported Faulty/In Repair, or the Logistics and Resources Officer records a crew member reserved for it (including its Field Commander) as Unavailable, the system shall, in one transaction, set the resource's new status, release it from the task, close its assignment record with an end time (keeping it for cost and utilization calculations) and mark the package item as missing, keep the task Active with its other resources Reserved, and after commit alert every Dispatcher that a replacement is required for the task. |
| **FR-13** | For a missing package item of an Active task, the system shall allow a Dispatcher either to reserve an Available resource of the same crew role or equipment type, selected manually or automatically under the rules of FR-02, FR-06 and FR-07, with the task remaining Active, or to confirm that the item is no longer needed, except for the Field Commander item, which must be replaced. When a replacement Field Commander is reserved, the system shall move the task's permissions (viewing its details, reporting faults, reporting completion) to the new Field Commander and remove them from the previous one. The system shall fill or waive each missing item at most once. |

### 3.2 OP-2 – Produce the SLA report

| ID | Requirement |
| :--- | :--- |
| **FR-14** | The system shall automatically generate and store the SLA Compliance Report for the previous calendar month at 00:05 server time on the first day of every month. |
| **FR-15** | The SLA Compliance Report shall show, for each of the 5 incident types, the number of qualifying incidents and the mean time from incident opening to the last Completed task; cancelled tasks are ignored (D-07, D-08, lecturer answer 4). The mean shall be formatted as hh:mm. |
| **FR-16** | The system shall include in the SLA Compliance Report for month M only incidents that qualify according to D-08 (status Closed, closing timestamp within M, at least one task Completed). |
| **FR-17** | The system shall allow the System Administrator to generate the SLA Compliance Report on demand for any completed past calendar month, and shall reject the current month or a future month with an explanatory message. |
| **FR-18** | For an incident type with no qualifying incidents in the month, the system shall display a count of 0 and a mean of "N/A" instead of omitting the type. |
| **FR-19** | The system shall store every generated SLA Compliance Report and shall make stored reports viewable only to users with the System Administrator role. |

---

## 4. Non-Functional Requirements (NFR)

| ID | Category | Requirement (measurable threshold) |
| :--- | :--- | :--- |
| **NFR-01** | Performance | A manual reservation of a package of up to 10 resources shall complete (commit or rollback) and its result shall be displayed to the Dispatcher within **2 seconds** for at least **95%** of requests, with **20** workstations connected over the LAN. |
| **NFR-02** | Performance | Every task or resource state change shall be pushed by the server to all connected workstations that display the affected incident within **1 second** after commit, with **0** client polling requests. |
| **NFR-03** | Performance | An automatic allocation, including all retries, shall complete with success or failure within **5 seconds** of the request (D-03), and while it runs at least **95%** of other clients' requests shall be answered within **2 seconds**. |
| **NFR-04** | Performance | An on-demand SLA Compliance Report for a month with up to **10,000** qualifying incidents shall be generated and displayed within **10 seconds**. |
| **NFR-05** | Security | All client–server traffic shall be encrypted with **TLS 1.2 or higher**, and user passwords shall be stored only as **bcrypt hashes with a cost factor of at least 12**. |
| **NFR-06** | Security | The server shall verify the user's role and task assignment on **100%** of reservation, completion and report requests, reject every unauthorized request, and write an audit-log entry for the rejection within **1 second**. |
| **NFR-07** | Reliability | When **50** reservation requests for the same resource are submitted simultaneously from different workstations, exactly **1** shall succeed, and **0** double reservations shall occur across **1,000** repeated test runs. |
| **NFR-08** | Reliability | A manual reservation or replacement that cannot complete within the **5-second** limit of D-03 shall be rolled back with nothing reserved, and the Dispatcher shall be informed within **6 seconds** of confirming. |
| **NFR-09** | Reliability | If a client disconnects after sending a completion report, the server shall commit either all effects of FR-09 or none of them, and the reconnected client shall display the committed task state within **3 seconds** of reconnecting. |
| **NFR-10** | Reliability | If the server is not running at the scheduled monthly generation time, it shall generate every missing monthly SLA Compliance Report within **5 minutes** after start-up, exactly **once** per month. |
| **NFR-11** | Usability | A trained Dispatcher shall reserve a task's package by automatic allocation in at most **3 user actions** from the incident status screen, and a Field Commander shall report task completion in at most **2 user actions**. |
| **NFR-12** | Usability | The System Administrator shall generate an on-demand SLA Compliance Report in at most **4 user actions** from the main screen. |

---

## 5. Requirements Traceability Matrix (RTM)

| Req ID | Type | Operation | Requirement (short) | Mapped Use Case | Test ID |
| :--- | :-: | :-: | :--- | :--- | :--- |
| **FR-01** | FR | OP-1 | Dispatcher only; task must be Ready | UC-23 Reserve Resource Package | TEST-01, TEST-13, TEST-14 |
| **FR-02** | FR | OP-1 | Resources and Active transition in one locked transaction | UC-23 Reserve Resource Package, UC-24 Auto-Allocate Resource Package, UC-25 Reserve Resources Atomically | TEST-01, TEST-04 |
| **FR-03** | FR | OP-1 | Manual reservation or replacement: rollback, message and screen refresh | UC-23 Reserve Resource Package, UC-28 Assign Replacement Resource, UC-26 Roll Back Reservation | TEST-02 |
| **FR-04** | FR | OP-1 | Reject reserved Field Commander / unavailable crew / faulty item | UC-23 Reserve Resource Package, UC-25 Reserve Resources Atomically | TEST-03 |
| **FR-05** | FR | OP-1 | Push update after commit | UC-23 Reserve Resource Package | TEST-01 |
| **FR-06** | FR | OP-1 | Auto-allocation by criterion | UC-24 Auto-Allocate Resource Package | TEST-05 |
| **FR-07** | FR | OP-1 | Up to 2 alternative attempts, replace only taken resources, two failure messages | UC-24 Auto-Allocate Resource Package, UC-27 Retry with Next Best Alternative | TEST-06, TEST-07, TEST-15 |
| **FR-08** | FR | OP-1 | Only the assigned Field Commander completes an Active task with no missing item | UC-29 Report Task Completion | TEST-08, TEST-09, TEST-17 |
| **FR-09** | FR | OP-1 | Complete task, release resources, promote dependents | UC-29 Report Task Completion, UC-21 Release Task Resources, UC-30 Update Task Readiness | TEST-08, TEST-17 |
| **FR-10** | FR | OP-1 | Audit entries for reservation actions | UC-25 Reserve Resources Atomically, UC-29 Report Task Completion | TEST-01, TEST-02, TEST-06, TEST-08 |
| **FR-11** | FR | OP-1 | One package per task; serialised with cancellation | UC-25 Reserve Resources Atomically, UC-19 Cancel Task | TEST-14 |
| **FR-12** | FR | OP-1 | Lost resource released; task stays Active; Dispatcher alerted | UC-12 Report Equipment Fault, UC-09 Update Crew Member Availability | TEST-16 |
| **FR-13** | FR | OP-1 | Replace a missing item (manual or auto) or waive it; FC must be replaced and gets the task's permissions | UC-28 Assign Replacement Resource | TEST-16, TEST-17 |
| **FR-14** | FR | OP-2 | Automatic monthly generation | UC-34 Generate Monthly Reports | TEST-18 |
| **FR-15** | FR | OP-2 | Count and mean time to the last Completed task per incident type | UC-35 Generate SLA Compliance Report | TEST-18 |
| **FR-16** | FR | OP-2 | Qualifying-incident rule | UC-35 Generate SLA Compliance Report | TEST-20 |
| **FR-17** | FR | OP-2 | On-demand past month; reject current/future | UC-35 Generate SLA Compliance Report | TEST-19, TEST-21 |
| **FR-18** | FR | OP-2 | Show 0 / "N/A" for types without data | UC-35 Generate SLA Compliance Report | TEST-18, TEST-20 |
| **FR-19** | FR | OP-2 | Store reports; Administrator-only access | UC-06 View Management Reports | TEST-18, TEST-23 |
| **NFR-01** | NFR | OP-1 | Performance – reservation ≤ 2 s (95%) | UC-23 Reserve Resource Package | TEST-12 |
| **NFR-02** | NFR | OP-1 | Performance – push ≤ 1 s, no polling | UC-23 Reserve Resource Package, UC-29 Report Task Completion, UC-28 Assign Replacement Resource | TEST-01, TEST-08, TEST-16 |
| **NFR-03** | NFR | OP-1 | Performance – auto-allocation ≤ 5 s, others ≤ 2 s (95%) | UC-24 Auto-Allocate Resource Package | TEST-05, TEST-06, TEST-15 |
| **NFR-04** | NFR | OP-2 | Performance – report ≤ 10 s for 10,000 incidents | UC-35 Generate SLA Compliance Report | TEST-19 |
| **NFR-05** | NFR | OP-1, OP-2 | Security – TLS ≥ 1.2, bcrypt cost ≥ 12 | UC-01 Log In | TEST-11 |
| **NFR-06** | NFR | OP-1, OP-2 | Security – 100% server-side authorization | UC-23 Reserve Resource Package, UC-29 Report Task Completion, UC-06 View Management Reports, UC-35 Generate SLA Compliance Report | TEST-09, TEST-13, TEST-23 |
| **NFR-07** | NFR | OP-1 | Reliability – 0 double reservations | UC-23 Reserve Resource Package, UC-25 Reserve Resources Atomically | TEST-02 |
| **NFR-08** | NFR | OP-1 | Reliability – manual lock timeout 5 s with rollback (reservation and replacement) | UC-23 Reserve Resource Package, UC-25 Reserve Resources Atomically, UC-28 Assign Replacement Resource | TEST-04 |
| **NFR-09** | NFR | OP-1 | Reliability – atomic completion on disconnect | UC-29 Report Task Completion | TEST-10 |
| **NFR-10** | NFR | OP-2 | Reliability – catch-up within 5 min, exactly once | UC-34 Generate Monthly Reports | TEST-22 |
| **NFR-11** | NFR | OP-1 | Usability – ≤ 3 / ≤ 2 user actions | UC-24 Auto-Allocate Resource Package, UC-29 Report Task Completion | TEST-05, TEST-08 |
| **NFR-12** | NFR | OP-2 | Usability – ≤ 4 user actions | UC-35 Generate SLA Compliance Report | TEST-19 |

---

## 6. BDD Acceptance Tests (Given–When–Then)

**Shared test data (OP-1).** Incident **INC-101** "Haifa Port Building Collapse" (Building Collapse) is Open. Task **T-3** "Debris Removal" requires a package of: 1 Field Commander, 1 Heavy Equipment Operator, 1 Bulldozer, 1 Paramedic. Pool: Field Commander **CR-104** Dan Levi (user `dlevi`), Heavy Equipment Operator **CR-221**, Paramedic **CR-310** Noa Katz (user `crew7`), Bulldozers **EQ-07** (2 km from INC-101) and **EQ-12** (15 km from INC-101). Dispatchers: `disp1`, `disp2`.

### 6.1 OP-1 – Reserve a resource package and report task completion

### Acceptance Test [TEST-01]: Manual package reservation (Happy Path)
* **Given:** `disp1` is logged in as Dispatcher and is viewing the status screen of INC-101, with 20 workstations connected.
* **And:** T-3 is in state Ready, and CR-104, CR-221, CR-310 and EQ-07 are Available.
* **When:** `disp1` selects CR-104, CR-221, CR-310 and EQ-07 for T-3 and confirms the reservation.
* **Then:** CR-104, CR-221, CR-310 and EQ-07 are Reserved for T-3, and T-3 is in state Active.
* **And:** the result is displayed to `disp1` within 2 seconds.
* **And:** every other workstation that displays INC-101 shows T-3 as Active within 1 second, without having sent a polling request.
* **And:** the audit log contains one entry per reserved resource with user `disp1` and the timestamp.

### Acceptance Test [TEST-02]: Two Dispatchers reserve the same bulldozer at the same moment (Exception – concurrency)
* **Given:** `disp1` is reserving EQ-07 for T-3 of INC-101 (Haifa), and `disp2` is reserving EQ-07 together with crew member CR-330 for task T-7 of INC-102 (Tel Aviv).
* **And:** EQ-07 and CR-330 are Available, and both T-3 and T-7 are Ready.
* **When:** both Dispatchers confirm their reservations at the same moment.
* **Then:** exactly one of the two reservations commits and EQ-07 is Reserved for exactly one task.
* **And:** the other transaction is rolled back: CR-330 is still Available if T-7 lost, and the losing task remains Ready.
* **And:** the losing Dispatcher sees "EQ-07 Bulldozer is already reserved for task T-3." if T-7 lost, or "EQ-07 Bulldozer is already reserved for task T-7." if T-3 lost, and their screen is refreshed to show EQ-07 as Reserved.
* **And:** the audit log contains the successful reservation and the failed attempt with their users and timestamps.
* **And:** when the scenario is repeated 1,000 times with 50 simultaneous requests for the same resource, 0 double reservations occur.

### Acceptance Test [TEST-03]: Reservation rejected because a package resource is no longer eligible (Exception)
* **Given:** `disp1` is viewing INC-101.
* **And:** T-3 is Ready and CR-104, CR-221, CR-310 and EQ-07 are Available.
* **When:** `disp1` selects CR-104, CR-221, CR-310 and EQ-07 for T-3.
* **And:** before `disp1` confirms, `<condition>`.
* **And:** `disp1` confirms.
* **Then:** no resource of the package is reserved and the state of T-3 is unchanged.
* **And:** `disp1` sees the message `<message>`.

| condition | message |
| :--- | :--- |
| CR-104 (Field Commander) becomes Reserved for task T-1 (Active) | "CR-104 Dan Levi is already reserved for task T-1." |
| EQ-07 (Bulldozer) becomes Faulty/In Repair | "EQ-07 Bulldozer is Faulty/In Repair." |
| CR-310 (Paramedic) becomes Unavailable (sick leave) | "CR-310 Noa Katz is unavailable." |

### Acceptance Test [TEST-04]: Record locks are not obtained in time (Exception – timeout)
* **Given:** `disp1` is viewing INC-101 and `<situation>`.
* **And:** a test database session holds a row lock on `<locked>` for 10 seconds.
* **When:** `disp1` `<action>`.
* **Then:** the transaction is rolled back after 5 seconds.
* **And:** `<unchanged>`.
* **And:** `disp1` sees "The reservation could not be completed in time; nothing was reserved. Please retry." within 6 seconds of confirming.
* **And:** after the test session releases its lock, `disp1` retries and the `<operation>` succeeds.

| situation | locked | action | unchanged | operation |
| :--- | :--- | :--- | :--- | :--- |
| T-3 is Ready and all package resources are Available | EQ-07 | confirms the reservation of CR-104, CR-221, CR-310 and EQ-07 for T-3 | no resource is reserved and T-3 remains Ready | reservation |
| T-3 is Active, its Bulldozer item is missing because EQ-07 was reported Faulty/In Repair, and EQ-12 is Available | EQ-12 | selects EQ-12 manually as the replacement for the Bulldozer item of T-3 and confirms | EQ-12 is not Reserved, T-3 remains Active and its Bulldozer item is still missing | replacement |

### Acceptance Test [TEST-05]: Automatic allocation by proximity (Happy Path)
* **Given:** `disp1` is viewing INC-101, T-3 is Ready, and EQ-07 (2 km) and EQ-12 (15 km) are both Available.
* **And:** for each required crew role at least one crew member is Available.
* **When:** `disp1` selects T-3, chooses "Auto-Allocate" and confirms the default criterion "Nearest to incident".
* **Then:** EQ-07 and the nearest Available crew member of each required role are Reserved for T-3, and T-3 is Active.
* **And:** the allocation completes within 5 seconds.
* **And:** `disp1` performed no more than 3 user actions.

### Acceptance Test [TEST-06]: Two automatic allocations compete for the same bulldozer (Exception – concurrency recovery)
* **Given:** task T-3 of INC-101 and task T-8 of INC-103 are both Ready and each requires one Bulldozer.
* **And:** EQ-07 is the best-ranked Available bulldozer for both tasks and EQ-12 is the second best for both.
* **When:** `disp1` and `disp2` request automatic allocation for T-3 and T-8 at the same moment.
* **Then:** one task receives EQ-07 and the other task receives EQ-12, and both tasks are Active.
* **And:** the task that received EQ-12 keeps the crew members selected in its first attempt; only the Bulldozer was replaced.
* **And:** neither Dispatcher had to repeat the request.
* **And:** both allocations complete within 5 seconds.
* **And:** during the allocations, a third workstation that opens the incident list receives a response within 2 seconds.
* **And:** the audit log contains exactly one retry entry.

### Acceptance Test [TEST-07]: Automatic allocation fails when no alternative exists (Exception)
* **Given:** T-3 of INC-101 and T-8 of INC-103 are Ready and each requires one Bulldozer.
* **And:** EQ-07 is the only Available bulldozer in the pool.
* **When:** `disp1` and `disp2` request automatic allocation for T-3 and T-8 at the same moment.
* **Then:** exactly one of the tasks is Active with EQ-07.
* **And:** for the other task no resource is reserved, the task remains Ready, and its Dispatcher sees "Auto-allocation failed: no Available Bulldozer in the pool. Nothing was reserved."

### Acceptance Test [TEST-08]: Field Commander reports task completion (Happy Path)
* **Given:** T-3 is Active with CR-104, CR-221, CR-310 and EQ-07 Reserved, and `dlevi` (CR-104) is logged in as its Field Commander.
* **And:** task T-5 is Blocked with prerequisites T-3 and T-4, where T-4 is Completed.
* **And:** task T-6 is Blocked with prerequisites T-3 and T-9, where T-9 is Active.
* **When:** `dlevi` selects "Report Completion" on T-3 and confirms.
* **Then:** T-3 is Completed and its completion timestamp is recorded.
* **And:** CR-104, CR-221, CR-310 and EQ-07 are Available.
* **And:** T-5 is Ready and T-6 remains Blocked.
* **And:** every workstation that displays INC-101 shows these changes within 1 second.
* **And:** the audit log contains the completion and the state transitions with user `dlevi`.
* **And:** `dlevi` performed no more than 2 user actions.

### Acceptance Test [TEST-09]: Completion reported by a Field Commander who is not assigned (Exception – authorization)
* **Given:** `rmizrahi` is logged in as a Field Commander who commands task T-4 of INC-101 but not T-3.
* **And:** T-3 is Active with `dlevi` as its Field Commander.
* **When:** a completion request for T-3 is sent from the session of `rmizrahi`.
* **Then:** the server rejects the request, and T-3 remains Active with all its resources Reserved.
* **And:** `rmizrahi` sees "Only the assigned Field Commander can complete this task."
* **And:** the rejected attempt is written to the audit log within 1 second.

### Acceptance Test [TEST-10]: Connection lost while reporting completion (Exception – recovery)
* **Given:** T-3 is Active and `dlevi` is its logged-in Field Commander.
* **When:** `dlevi` confirms completion of T-3 and the client's network connection drops before the server's response arrives.
* **Then:** on the server, either T-3 is Completed with all its resources Available and dependents promoted, or T-3 is Active with no change; no partial state exists.
* **And:** when the client reconnects, it displays the committed state of T-3 within 3 seconds.
* **And:** if `dlevi` sends the completion again and T-3 is already Completed, the system makes no further change and displays "The task is already Completed."

### Acceptance Test [TEST-11]: Encrypted traffic and hashed passwords (Security)
* **Given:** a packet capture is running on the LAN segment between a client workstation and the server.
* **When:** `disp1` logs in and reserves a package for T-3.
* **Then:** the capture shows a TLS 1.2 or TLS 1.3 session and no plaintext username, password or reservation data.
* **And:** the users table stores `disp1`'s password only as a bcrypt hash with a cost factor of 12 or higher.

### Acceptance Test [TEST-12]: Manual reservation response time under load (Performance)
* **Given:** 20 Dispatcher workstations are connected to the server over the LAN.
* **And:** the database contains 1,000 Ready tasks and enough Available resources for their packages (4 to 10 resources each), and 100 of the planned requests deliberately compete for the same resources.
* **When:** the 20 workstations submit the 1,000 manual reservations within 10 minutes.
* **Then:** at least 950 reservations (95%) have their result (commit or rollback message) displayed within 2 seconds of confirmation.
* **And:** no resource is Reserved for more than one task.

### Acceptance Test [TEST-13]: Reservation requested by a user who is not a Dispatcher (Exception – authorization)
* **Given:** T-3 of INC-101 is Ready and all its package resources are Available.
* **And:** `<user>` is logged in with the role `<role>`.
* **When:** a reservation request for T-3 is sent from the session of `<user>`.
* **Then:** the server rejects the request, no resource is reserved and T-3 remains Ready.
* **And:** `<user>` sees "Only a Dispatcher can reserve resources."
* **And:** the rejected attempt is written to the audit log within 1 second.

| user | role |
| :--- | :--- |
| `dlevi` | Field Commander |
| `log1` | Logistics and Resources Officer |
| `crew7` | Field Crew Member |
| `admin1` | System Administrator |

### Acceptance Test [TEST-14]: Competing requests on the same task (Exception – concurrency)
* **Given:** T-3 of INC-101 is Ready.
* **And:** package A (CR-104, CR-221, CR-310, EQ-07) and package B (CR-105, CR-222, CR-311, EQ-12) are all Available, and each fits T-3.
* **When:** `disp1` confirms the reservation of package A for T-3 and, at the same moment, `disp2` `<competing action>`.
* **Then:** `<expected result>`.
* **And:** no resource is Reserved for T-3 unless T-3 is Active, and T-3 never has more than one reserved package.

| competing action | expected result |
| :--- | :--- |
| confirms the reservation of package B for T-3 | Exactly one package is Reserved and T-3 is Active; the other Dispatcher sees "Task T-3 is no longer Ready." and no resource of the other package is Reserved |
| cancels T-3 | T-3 is Cancelled and all 8 resources are Available. If the cancellation locked INC-101 first, the reservation is rejected with "Task T-3 is no longer Ready."; if the reservation committed first, the cancellation releases package A |

### Acceptance Test [TEST-15]: Automatic allocation finishes within 5 seconds while locks are held (Exception – timeout and performance)
* **Given:** T-3 of INC-101 is Ready and needs one Bulldozer, EQ-07, EQ-12 and EQ-15 are Available bulldozers ranked 1st, 2nd and 3rd by distance, and all other package items are Available.
* **And:** a test database session holds row locks on EQ-07 and EQ-12 for 30 seconds.
* **And:** 19 other workstations send 200 ordinary requests (for example, opening incident status screens) during the test.
* **When:** `disp1` requests automatic allocation for T-3.
* **Then:** within 5 seconds of the request, T-3 is Active with EQ-15 after 2 retries (the lock timeouts on EQ-07 and EQ-12 each count as a lost attempt), and EQ-07 and EQ-12 are unchanged.
* **And:** at least 95% of the 200 other requests are answered within 2 seconds.
* **And:** when the test is repeated with EQ-15 also locked, the third attempt also loses to contention and the allocation fails within 5 seconds with "Auto-allocation failed: resources were lost to competing requests. Nothing was reserved.", nothing is reserved and T-3 remains Ready.

### Acceptance Test [TEST-16]: A package resource is lost during an Active task and replaced (Exception – recovery)
* **Given:** T-3 of INC-101 is Active with CR-104 (Field Commander, `dlevi`), CR-221 (Heavy Equipment Operator), CR-310 (Paramedic) and EQ-07 (Bulldozer) Reserved, and `disp1` and `disp2` are viewing INC-101.
* **And:** Field Commanders CR-105 (user `ycohen`) and CR-106 (user `aperetz`) are Available.
* **When:** `log1` records sick leave for CR-104.
* **Then:** CR-104 has status Unavailable, is no longer Reserved for T-3, and its assignment record for T-3 is closed with the time of the event.
* **And:** T-3 is still Active, its other three resources are still Reserved, and its Field Commander item is shown as missing.
* **And:** every Dispatcher workstation shows a "Replacement required" alert for T-3 within 1 second.
* **When:** `disp1` fills the missing item with CR-105 manually, while `disp2` manually confirms CR-106 for the same item at the same moment.
* **Then:** exactly one of CR-105 and CR-106 is Reserved for T-3, the other one is still Available, T-3 is still Active and the alert is resolved.
* **And:** the other Dispatcher sees "This item has already been handled." and nothing else is reserved.
* **And:** the user of the reserved Field Commander (`ycohen` for CR-105, `aperetz` for CR-106) is T-3's assigned Field Commander: only this Field Commander sees T-3's resources and can report a fault or completion for T-3.
* **And:** the cost summary and the utilization report count CR-104 for T-3 up to the end time of its closed assignment record.

### Acceptance Test [TEST-17]: Completion is blocked while a package item is missing (Exception)
* **Given:** T-3 of INC-101 is Active with CR-104, CR-221, CR-310 and EQ-07 Reserved, and `dlevi` (CR-104) is its logged-in Field Commander.
* **And:** EQ-07 was reported Faulty/In Repair, so the Bulldozer item of T-3 is missing.
* **When:** `dlevi` selects "Report Completion" on T-3 and confirms.
* **Then:** the server rejects the request, T-3 remains Active, and CR-104, CR-221 and CR-310 remain Reserved.
* **And:** `dlevi` sees "Task T-3 cannot be completed: the Bulldozer item has no resource. The Dispatcher must assign a replacement or confirm that it is no longer needed."
* **When:** `disp1` confirms that the Bulldozer item of T-3 is no longer needed, and `dlevi` reports completion of T-3 again.
* **Then:** T-3 is Completed, CR-104, CR-221 and CR-310 are Available, and EQ-07 remains Faulty/In Repair.
* **And:** in a second run where CR-104 is recorded as Unavailable instead, `disp1` selects "No Longer Needed" on the Field Commander item and sees "A task must have a Field Commander. Assign a replacement.", and the item stays missing.

### 6.2 OP-2 – Produce the SLA report

### Acceptance Test [TEST-18]: Automatic monthly SLA report (Happy Path)
* **Given:** no SLA Compliance Report exists for October 2026.
* **And:** the following incidents were finally closed in October 2026 and qualify for it:

| Incident | Type | Opened | Last task completed | Closed | Time |
| :--- | :--- | :--- | :--- | :--- | :--- |
| INC-201 | Fire | 2026-10-03 08:00 | 2026-10-03 12:30 | 2026-10-03 15:00 | 04:30 |
| INC-202 | Fire | 2026-10-10 22:00 | 2026-10-11 01:30 | 2026-10-11 09:00 | 03:30 |
| INC-203 | Road Accident | 2026-10-15 09:00 | 2026-10-15 10:15 | 2026-10-16 08:00 | 01:15 |

* **When:** the server clock reaches 2026-11-01 00:05.
* **Then:** the SLA Compliance Report for October 2026 is stored.
* **And:** it shows Fire: 2 incidents, mean 04:00; Road Accident: 1 incident, mean 01:15.
* **And:** it shows Building Collapse, Hazmat Leak and Natural Disaster each with 0 incidents and mean "N/A".
* **And:** the report is listed under Reports for the System Administrator, and the generation is recorded in the audit log.

### Acceptance Test [TEST-19]: On-demand SLA report for a past month (Happy Path – performance and usability)
* **Given:** `admin1` is logged in as System Administrator on the main screen.
* **And:** September 2026 contains 10,000 qualifying incidents.
* **When:** `admin1` selects "Reports", selects "Generate SLA Compliance Report", selects September 2026 and selects "Generate".
* **Then:** the SLA Compliance Report for September 2026 is displayed with one row for each of the 5 incident types within 10 seconds.
* **And:** `admin1` performed no more than 4 user actions.

### Acceptance Test [TEST-20]: Non-qualifying incidents are excluded (Exception – edge data)
* **Given:** the following incidents exist:

| Incident | Type | Situation |
| :--- | :--- | :--- |
| INC-204 | Fire | Cancelled on 2026-10-05 |
| INC-205 | Hazmat Leak | Opened 2026-10-20; one task is still Active, so the incident is Open |
| INC-206 | Natural Disaster | Opened 2026-10-22; all tasks Cancelled, none Completed; Closed 2026-10-23 |
| INC-207 | Building Collapse | Opened 2026-09-30 20:00; last task completed 2026-09-30 23:00; Closed 2026-10-01 09:00 |
| INC-208 | Road Accident | Opened 2026-10-25 08:00; task A Completed 10:00; task B Cancelled 12:00; Closed 2026-10-25 14:00 |
| INC-209 | Fire | Opened 2026-10-31 10:00; last task completed 2026-10-31 16:00; In Settlement at midnight; Closed 2026-11-01 08:00 |
| INC-210 | Hazmat Leak | Opened 2026-10-12 07:00; last task completed 2026-10-12 09:00; settlement rejected 2026-10-12 11:00, Open since then |

* **And:** no other incidents qualify for October 2026.
* **When:** `admin1` generates the SLA Compliance Report for October 2026.
* **Then:** Building Collapse shows 1 incident with mean 03:00 (INC-207 counts in October, the month it was closed, and is measured to its last task completion, not to the closing).
* **And:** Road Accident shows 1 incident with mean 02:00 (INC-208 is measured to its last Completed task at 10:00, not to the cancellation at 12:00).
* **And:** Fire, Hazmat Leak and Natural Disaster show 0 incidents with mean "N/A" (INC-209 was closed in November; INC-205 and INC-210 are Open; INC-206 has no Completed task).

### Acceptance Test [TEST-21]: Report requested for an invalid month (Exception)
* **Given:** the server date is 2026-11-15 and `admin1` is logged in as System Administrator.
* **When:** `admin1` requests the SLA Compliance Report for `<month>`.
* **Then:** the system displays "Reports can be generated only for completed months."
* **And:** no report is stored.

| month |
| :--- |
| November 2026 (current month) |
| December 2026 (future month) |

### Acceptance Test [TEST-22]: Server was down at month end (Exception – recovery)
* **Given:** the server was stopped on 2026-10-31 at 23:00 and no SLA Compliance Report exists for October 2026.
* **When:** the server is started on 2026-11-01 at 09:00.
* **Then:** the SLA Compliance Report for October 2026 is generated and stored by 09:05.
* **And:** exactly one SLA Compliance Report exists for October 2026.
* **And:** when the server is restarted again at 09:30, no additional SLA Compliance Report for October 2026 is generated.

### Acceptance Test [TEST-23]: Non-administrator tries to access the SLA report (Exception – authorization)
* **Given:** `disp1` is logged in as Dispatcher.
* **When:** a request to view or generate the SLA Compliance Report is sent from the session of `disp1`.
* **Then:** the server rejects the request and returns no report data.
* **And:** the Reports menu is not displayed in the Dispatcher interface.
* **And:** the rejected attempt is written to the audit log within 1 second.
