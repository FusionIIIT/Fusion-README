# Fusion ERP Modules - Summary

---

## Production Modules (00_*)

These modules are already in production.

---

### 00_Programme_Curriculum_Integration

**What this module is:** The Programme Curriculum module defines the academic structure - programmes (BTech, MTech, PhD, ...), disciplines (CSE, ECE, ME, ...), batches (2021, 2022, ...), semesters, courses, and course-instructor mappings.

**Why it exists:** This is the **foundation** of academic data. Before any student registers, before any grade is given, the curriculum must exist. It defines what programmes are offered, what courses exist, who teaches them.

**Key tables:** Programme, Discipline, Batch, Semester, Course, CourseSlot, CourseInstructor

---

### 00_Academic_Information_Integration

**What this module is:** The Academic Information module stores student identity and academic status - the Student table with roll numbers, batch assignments, and current academic standing.

**Why it exists:** This is the **student master data**. Every student in Fusion has a record here. Other modules (registration, grades, dues) reference this table to identify students.

**Key tables:** Student

---

### 00_Academic_Procedures_Integration

**What this module is:** The Academic Procedures module handles course registration, pre-registration, add/drop, thesis registration, and dues management.

**Why it exists:** This is where students **interact with their academic journey**. They register for courses, handle dues, apply for procedures. The course_registration table is central to knowing who is enrolled where.

**Key tables:** course_registration, InitialRegistration, FinalRegistration, Dues, FeePayments, ThesisTopicProcess

---

### 00_Online_CMS_Integration

**What this module is:** The Online CMS module stores student grades for each course - the Student_grades table.

**Why it exists:** Grades are the **academic outcome**. After courses are taught and exams conducted, grades are stored here. Other modules query this for backlog checks, eligibility verification, and academic standing.

**Key tables:** Student_grades

---

## New Modules to Build (01-23)

These modules are **NOT in production**. They will be built from scratch but will integrate with production data from the 00_* modules.

---

### 01_LMS_Integration (Learning Management System)

**What this module is:** The LMS will handle online course delivery - managing lecture materials, uploading resources, tracking attendance, conducting quizzes, handling assignments, and facilitating discussions.

**Why we need to build this:** Currently there's no integrated digital learning platform. Faculty share materials via email or external drives. Assignments are submitted physically. Quizzes happen on paper. This module will provide a unified online classroom.

**Key dependencies:** Student, course_registration, Courses, CourseInstructor, ExtraInfo

---

### 02_Award_Scholarship_Integration

**What this module is:** The Award and Scholarship module will manage all financial aid and recognition programs - merit scholarships, need-based aid, government scholarships, sports awards, and academic excellence recognitions.

**Why we need to build this:** Students receive various scholarships but currently eligibility is verified manually using spreadsheets. This module will automate the entire scholarship lifecycle - applications, eligibility checks against academic records, approvals, and disbursement tracking.

**Key dependencies:** Student, Batch, Programme, Student_grades, FeePayments, Dues

---

### 03_Department_Integration

**What this module is:** The Department module will handle department-level operations - managing faculty within a department, department announcements, research coordination, curriculum oversight, and department-specific student interactions.

**Why we need to build this:** Each academic department (CSE, ECE, ME, etc.) has administrative needs but currently HODs lack a unified view of their domain. This module will provide department-scoped views - their students, faculty workload, course offerings, and research activities in one place.

**Key dependencies:** Discipline, Batch, Student, Course, CourseInstructor, Faculty

---

### 04_Other_Academic_Procedure_Integration

**What this module is:** The Other Academic Procedure module will handle miscellaneous academic requests - bonafide certificates, thesis topic approval, assistantship claims, branch change applications, no-dues certificates, and special permissions.

**Why we need to build this:** Beyond course registration, students need various academic services. Currently bonafide certificates require paper applications, thesis approvals are tracked manually. This module will automate these workflows.

**Key dependencies:** Student, Batch, Faculty, Dues, BranchChange

---

### 05_File_Tracking_System_Integration

**What this module is:** The File Tracking System will manage document workflow across the institute - tracking files as they move between offices, managing approvals, and maintaining audit trails.

**Why we need to build this:** Files (leave applications, purchase requests, student petitions) move through multiple offices. Currently file location is often unknown causing delays. This module will provide real-time file tracking.

**Key dependencies:** Discipline, ExtraInfo, Faculty, Staff, DepartmentInfo

---

### 06_RSPC_Integration (Research, Sponsored Projects & Consultancy)

**What this module is:** The RSPC module will manage research activities - sponsored projects (funded research), consultancy services, patent filings, publications, and research funding applications.

**Why we need to build this:** Faculty and students conduct research that needs tracking. Currently sponsored projects are managed in spreadsheets. This module will provide unified project lifecycle management for the RSPC cell.

**Key dependencies:** Faculty, Student, Discipline, ExtraInfo

---

### 07_PS_Management_Integration (Purchase & Store)

**What this module is:** The Purchase and Store Management module will handle procurement - purchase requests (indents), vendor management, purchase orders, goods receipt, inventory tracking, and store operations.

**Why we need to build this:** Institutes purchase equipment, consumables, and services. Currently indents are paper-based, vendor records scattered. This module will digitize the entire procurement workflow.

**Key dependencies:** Discipline, ExtraInfo, Faculty, Staff, DepartmentInfo

---

### 08_HR_EIS_Integration (Employee Information System)

**What this module is:** The HR/EIS module will manage employee lifecycle - from joining to retirement. It will track service records, qualifications, performance appraisals, training, promotions, leave, and all employee-related administration.

**Why we need to build this:** Currently HR processes are paper-based - leave applications, appraisals, service records. This module will provide digital employee management with self-service capabilities.

**Key dependencies:** ExtraInfo, Faculty, Staff, Discipline, CourseInstructor

---

### 09_PHC_Integration (Primary Health Center)

**What this module is:** The PHC module will manage the institute's health center - patient records, consultations, prescriptions, medical history, ambulance services, and health insurance claims.

**Why we need to build this:** Currently medical records are paper-based, making history tracking difficult. This module will digitize health center operations - appointments, prescriptions, medical history.

**Key dependencies:** Student, ExtraInfo, Faculty, Staff

---

### 10_Visitor_Hostel_Integration

**What this module is:** The Visitor Hostel module will manage the institute's guest house - room bookings, visitor registration, billing, and accommodation for official guests.

**Why we need to build this:** Currently booking is via phone/email, room availability is checked manually. This module will provide online room booking, automated availability checks, and digital billing.

**Key dependencies:** ExtraInfo, Faculty, Staff, Discipline

---

### 11_Hostel_Management_Integration

**What this module is:** The Hostel Management module will handle student accommodation - room allocation, hostel transfers, leave requests, visitor entry, complaints, and hostel administration.

**Why we need to build this:** Currently room allocation is manual, leave requests are on paper. This module will provide automated room allocation based on batch/programme/gender, digital leave requests, and warden dashboards.

**Key dependencies:** Student, Batch, ExtraInfo, Dues (hostel_due)

---

### 12_Mess_Management_Integration

**What this module is:** The Mess Management module will handle dining operations - menu planning, student mess registration, meal tracking, billing, rebates for missed meals, and feedback collection.

**Why we need to build this:** Currently mess registration is manual, rebate requests on paper. This module will digitize mess operations - online registration, automated billing, digital rebate workflows.

**Key dependencies:** Student, Batch, ExtraInfo, Dues (mess_due)

---

### 13_Placement_Cell_Integration

**What this module is:** The Placement Cell module will manage campus recruitment - company registrations, job postings, student placement profiles, application tracking, interview scheduling, and offer management.

**Why we need to build this:** Currently the placement process is fragmented - companies register via email, students submit profiles on paper, eligibility is checked manually. This module will digitize and automate the entire placement process.

**Key dependencies:** Student, Batch, Discipline, Programme, Student_grades, ExtraInfo

---

### 14_Gymkhana_Integration

**What this module is:** The Gymkhana module will manage student extracurricular life - clubs (technical, cultural, sports), societies, events, budgets, inventory, elections for student council, and activity registrations.

**Why we need to build this:** Currently club management is manual - member lists on paper, budgets tracked in spreadsheets. This module will provide digital club management, automated elections, and centralized event coordination.

**Key dependencies:** Student, Batch, ExtraInfo, Faculty

---

### 15_Complaint_Management_Integration

**What this module is:** The Complaint Management module will handle grievance redressal - complaint registration, categorization (academic/hostel/mess/infrastructure), assignment to handlers, tracking, and resolution.

**Why we need to build this:** Currently complaints are verbal or via scattered emails with no tracking. This module will provide structured complaint registration, automatic routing, and resolution tracking.

**Key dependencies:** Student, ExtraInfo, Faculty, Staff, DepartmentInfo

---

### 16_IWD_Integration (Institute Works Department)

**What this module is:** The IWD module will manage infrastructure - construction projects, maintenance requests, electrical/civil works, contractor management, work orders, and building/asset management.

**Why we need to build this:** Currently work requests are manual, tracking is on paper, contractor management is fragmented. This module will provide digital work order management and project tracking.

**Key dependencies:** ExtraInfo, Faculty, Staff, DepartmentInfo

---

### 17_System_Admin_Integration

**What this module is:** The System Administration module will manage user accounts, role assignments, permissions, system configurations, access controls, and audit logs.

**Why we need to build this:** Currently user management is scattered. This module will provide centralized account creation, role assignment (student/faculty/admin), and permission management.

**Key dependencies:** ExtraInfo, Student, Faculty, Staff, all module tables

---

### 18_Dashboards_Integration

**What this module is:** The Dashboards module will provide role-based summary views - students see their courses and grades, faculty see their advisees and classes, admins see analytics and metrics.

**Why we need to build this:** Users need a personalized landing page. Currently there's no unified dashboard. This module will aggregate data from all modules into role-based views.

**Key dependencies:** ALL production tables (Student, course_registration, Student_grades, Dues, ExtraInfo, etc.)

---

### 19_Backup_Archive_Integration

**What this module is:** The Backup & Archive module will manage data backups, archival of old records, retention policies, and disaster recovery procedures.

**Why we need to build this:** Currently there's no systematic backup strategy. This module will ensure data persistence, define retention policies, and enable disaster recovery.

**Key dependencies:** ALL tables (for backup/restore)

---

### 20_System_Security_Integration

**What this module is:** The System Security module will manage authentication, authorization, access control lists, security policies, threat monitoring, and compliance.

**Why we need to build this:** Currently security is basic. This module will enforce security policies, implement role-based access control, and detect unauthorized access.

**Key dependencies:** Django User model, ExtraInfo, ALL modules

---

### 21_Patent_Management_Integration

**What this module is:** The Patent Management module will track intellectual property - invention disclosures, patent applications, filing status, inventor details, and technology transfer.

**Why we need to build this:** Currently patent tracking is manual. This module will provide digital invention disclosure, application tracking, and IP portfolio management.

**Key dependencies:** ExtraInfo, Faculty, Student, DepartmentInfo

---

### 22_Visitor_Management_Integration

**What this module is:** The Visitor Management module will handle campus visitor registration, gate passes, entry/exit tracking, host verification, and visitor logs.

**Why we need to build this:** Currently visitor tracking is manual - visitors sign a register, hosts are called for verification. This module will provide digital visitor registration and real-time tracking.

**Key dependencies:** ExtraInfo, Faculty, Staff, DepartmentInfo

---

### 23_Internal_Audit_Integration

**What this module is:** The Internal Audit module will manage compliance audits, process reviews, audit scheduling, findings tracking, and corrective action monitoring.

**Why we need to build this:** Currently audits are conducted manually with findings tracked in documents. This module will provide structured audit scheduling and digital findings management.

**Key dependencies:** ExtraInfo, DepartmentInfo, ALL modules (for audit data)

---

## Summary Table

| # | Module | Status | Primary Purpose |
|---|--------|--------|-----------------|
| 00 | Programme Curriculum |  Production | Academic structure (programmes, courses) |
| 00 | Academic Information |  Production | Student master data |
| 00 | Academic Procedures |  Production | Course registration, dues |
| 00 | Online CMS |  Production | Student grades |
| 01 | LMS |  To Build | Online learning platform |
| 02 | Award Scholarship |  To Build | Scholarship management |
| 03 | Department |  To Build | Department-level views |
| 04 | Other Academic Procedure |  To Build | Certificates, misc requests |
| 05 | File Tracking |  To Build | Document workflow |
| 06 | RSPC |  To Build | Research project management |
| 07 | P&S Management |  To Build | Procurement & inventory |
| 08 | HR/EIS |  To Build | Employee management |
| 09 | PHC |  To Build | Health center operations |
| 10 | Visitor Hostel |  To Build | Guest house booking |
| 11 | Hostel Management |  To Build | Student hostel operations |
| 12 | Mess Management |  To Build | Dining operations |
| 13 | Placement Cell |  To Build | Campus recruitment |
| 14 | Gymkhana |  To Build | Clubs & extracurricular |
| 15 | Complaint Management |  To Build | Grievance redressal |
| 16 | IWD |  To Build | Infrastructure management |
| 17 | System Admin |  To Build | User & permission management |
| 18 | Dashboards |  To Build | Role-based summary views |
| 19 | Backup Archive |  To Build | Data backup & recovery |
| 20 | System Security |  To Build | Security & access control |
| 21 | Patent Management |  To Build | IP tracking |
| 22 | Visitor Management |  To Build | Campus visitor tracking |
| 23 | Internal Audit |  To Build | Compliance audits |

---
