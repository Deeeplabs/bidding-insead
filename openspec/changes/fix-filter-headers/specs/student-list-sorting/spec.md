## student-list-sorting

Fix sorting on student list tables so that all sortable column headers work without triggering SQL error 3065.

### Scenario: Sort students by Promotion column

- **Given** the admin is on the Student List page with students loaded
- **When** the admin clicks the "Promotion" column header to sort ascending
- **Then** the API returns students sorted by promotion label in ascending order
- **And** no SQL error 3065 occurs
- **And** the table displays the sorted results (not "No records found!")

### Scenario: Sort students by Programme column

- **Given** the admin is on the Student List page with students loaded
- **When** the admin clicks the "Programme" column header to sort ascending
- **Then** the API returns students sorted by programme name in ascending order
- **And** no SQL error 3065 occurs

### Scenario: Sort students by Home Campus column

- **Given** the admin is on the Student List page with students loaded
- **When** the admin clicks the "Home Campus" column header to sort ascending
- **Then** the API returns students sorted by campus label in ascending order
- **And** no SQL error 3065 occurs

### Scenario: Sort students by Name column

- **Given** the admin is on the Student List page
- **When** the admin clicks the "Name" column header to sort
- **Then** the frontend sends `sort=first_name` (not `full_name`) to the backend
- **And** the API returns students sorted by CONCAT(firstName, lastName)

### Scenario: Sort students by Peoplesoft ID column

- **Given** the admin is on the Student List page
- **When** the admin clicks the "Peoplesoft ID" column header to sort
- **Then** the frontend sends `sort=people_soft_id` (not `id`) to the backend
- **And** the API returns students sorted by student ID

### Scenario: Toggle sort direction cycles through ASC → DESC → default

- **Given** the admin is on the Student List page sorted by Promotion ASC
- **When** the admin clicks the "Promotion" column header again
- **Then** the sort direction changes to DESC
- **When** the admin clicks the "Promotion" column header a third time
- **Then** the sort resets to the default order (lastName ASC)

### Scenario: Sorting combined with filters does not return empty results

- **Given** the admin has applied a promotion filter on the Student List
- **When** the admin clicks any sortable column header
- **Then** the filtered results are re-sorted without losing the filter
- **And** the table shows the filtered and sorted data (not "No records found!")

### API Endpoint

- **Method**: POST `/api/students` (settings page) or GET `/api/students` (campaign student list)
- **Sort parameters**: `sort` (string field name) + `order` (asc/desc)
- **Affected sort fields**: `promotion_name`, `programme_name`, `home_campus`, `current_campus`
- **Response shape**: No changes — existing paginated student list response
