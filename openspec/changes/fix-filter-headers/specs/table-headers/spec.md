## table-headers

Standardize course and student table column headers across all admin dashboard views for consistency.

### Scenario: Student table headers are consistent across views

- **Given** the admin navigates to any page showing a student list table
- **When** the table renders
- **Then** the standard student column headers are:
  - **Name** (not "Full Name" — except in simulation/enrollment contexts where only full_name key is available)
  - **Email**
  - **Peoplesoft ID**
  - **Promotion**
  - **Programme**
  - **Credits** (not "Credit")
  - **Capital Left**
  - **Type**
  - **Home Campus**

### Scenario: Course table headers are consistent across dashboard views

- **Given** the admin navigates to any campaign dashboard course table
- **When** the table renders
- **Then** the standard course column headers are:
  - **Course** or **Course Name** — use "Course Name" consistently
  - **Section**
  - **Credits** (not "Credit")
  - **Seats Available** (not "Seat Available")
  - When present: **Bid Received**, **Enrolled**, **Waitlisted**, **Pending**, **Status**

### Scenario: Bidding Round course table uses standardized headers

- **Given** the admin is on the Bidding Round dashboard
- **When** the course table renders
- **Then** columns show: Course Name, Section, Credits, Seats Available, Bid Received, Status

### Scenario: Add-Drop course table uses standardized headers

- **Given** the admin is on the Add-Drop dashboard
- **When** the course table renders
- **Then** columns show: Course Name, Section, Credits, Seats Available, Total Bids Received, Enrolled, Waitlisted, Pending, Status

### Affected Components

**Student tables:**
- `bidding-admin/src/components/settings/student-setting-table.tsx` — main student settings table
- `bidding-admin/src/components/student/student-table-v2.tsx` — alternate student table
- `bidding-admin/src/components/dashboard/process/simulation/student-table-simulation.tsx`
- `bidding-admin/src/components/dashboard/process/final-enrollment/student-table-enrollment.tsx`

**Course tables:**
- `bidding-admin/src/components/dashboard/process/bidding-round/course-table-bidding-round.tsx` — "Seat Available" → "Seats Available"
- `bidding-admin/src/components/dashboard/process/add-drop/course-table.tsx` — "Credit" → "Credits"
- `bidding-admin/src/components/dashboard/process/pre-bidding/course-table-pre-bidding.tsx`
- `bidding-admin/src/components/settings/course-table-setting.tsx`
