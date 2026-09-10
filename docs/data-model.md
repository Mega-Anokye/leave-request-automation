# Data Model

Three data sources, one record per request.

## 1. Microsoft Form (input)

The form is set to **anyone can respond**, so it asks for the staff member's email
directly instead of relying on responder metadata (which returns `anonymous`).

| Field | Notes |
|---|---|
| Full Name | |
| Email | Used for all replies to the staff member |
| Employee ID | SAP employee code; the lookup key |
| Department | |
| Role | |
| Phone Number | |
| Zone | |
| Sub-Zone | |
| Leave Type | Choice |
| Leave Start Date | Date |
| Leave End Date | Date |

## 2. Staff Directory (Excel Online table: `StaffDirectory`)

The source of truth for who reports to whom. HR maintains it; the flow only reads it.

| Column | Used for |
|---|---|
| `SAP Code` | Match against the submitted Employee ID. **Note the space in the header**: expressions must read `item()?['SAP Code']` |
| `Name` | |
| `Zone` | |
| `Location` | |
| `JobTitle` | |
| `Dept` | |
| `Supervisor` | Written to the request record |
| `SupervisorEmail` | Approval is assigned here |
| `NotificationEmail` | Optional FYI recipients; may be blank or comma-separated |

Expressions reference headers **exactly as typed**, including spaces and case. When in
doubt, open the raw output of `List rows` from a test run and copy the property name
from there.

## 3. SharePoint list: `Leave Requests` (record of truth)

| Column | Type | Set by |
|---|---|---|
| Request Reference | Single line, indexed | Flow, at creation. The business key, e.g. `LR-20260724-4b7160` |
| Full Name, Email, Employee ID | Single line | Form |
| Department, Role, Phone Number | Single line | Form |
| Zone, Sub-Zone | Single line | Form |
| Leave Type | Choice | Form |
| Leave Start Date, Leave End Date | Date | Form (normalised to `yyyy-MM-dd`) |
| Number of Days | Number | Flow (inclusive count) |
| Supervisor Name, Supervisor Email | Single line | Directory lookup |
| Supervisor Decision | Choice | Approval outcome |
| Supervisor Comments | Multiple lines | Approval response |
| Supervisor Response Date | Date/time | Approval response |
| Status | Choice: `Record Not Found`, `Pending Supervisor Approval`, `Approved`, `Rejected` | Flow |
| Submission Date | Date/time | Flow (`utcNow()`) |
| HR Notified | Yes/No | Flow, in its own step after the HR email |

`Title` is not used as a key anywhere.
