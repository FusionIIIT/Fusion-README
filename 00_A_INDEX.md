# Fusion ERP - Module Integration Master Index

## Overview

This directory contains integration guides for Fusion ERP modules. Each guide explains how to sync module data with academic.

---

## Setting the system up

Start here if you are getting a machine running rather than looking up a table.

| Guide | What it covers |
|---|---|
| [QUICKSTART-macOS.md](QUICKSTART-macOS.md) | **Start here on a Mac** — the whole setup as one linear list of commands |
| [QUICKSTART.md](QUICKSTART.md) | **Start here on Ubuntu, WSL or Windows** — the whole setup as one linear list of commands |
| [Fusion_System_Setup.md](Fusion_System_Setup.md) | **The whole system, in order** — database, IAM, service tokens, Fusion-Integrated, Fusion-Academic |
| [Fusion_Database_Setup.md](Fusion_Database_Setup.md) | PostgreSQL and restoring the `fusionlab` dump (Windows, macOS, Linux) |

---

## Production Sync Modules

These modules need to sync with production database:

| # | Module | README File | Key Tables |
|---|--------|-------------|------------|
| 00 | Programme Curriculum | [00_Programme_Curriculum_Integration_README.md](00_Programme_Curriculum_Integration_README.md) | Programme, Discipline, Batch, Course, Semester, CourseSlot, CourseInstructor |
| 00 | Academic Information | [00_Academic_Information_Integration_README.md](00_Academic_Information_Integration_README.md) | Student |
| 00 | Academic Procedures | [00_Academic_Procedures_Integration_README.md](00_Academic_Procedures_Integration_README.md) | course_registration, FeePayments, Dues, BranchChange |
| 00 | Online CMS | [00_Online_CMS_Integration_README.md](00_Online_CMS_Integration_README.md) | Student_grades (only production table) |

---

## Quick Reference: Production Tables

### programme_curriculum

| Table | SQL Query | Purpose |
|-------|-----------|---------|
| Programme | `SELECT * FROM public.programme_curriculum_programme` | Degree programmes (UG/PG/PhD) |
| Discipline | `SELECT * FROM public.programme_curriculum_discipline` | Academic branches (CSE, ECE, etc.) |
| Batch | `SELECT * FROM public.programme_curriculum_batch` | Student batches per discipline |
| Curriculum | `SELECT * FROM public.programme_curriculum_curriculum` | Curriculum versions |
| Semester | `SELECT * FROM public.programme_curriculum_semester` | Semester definitions |
| Course | `SELECT * FROM public.programme_curriculum_course` | All courses |
| CourseSlot | `SELECT * FROM public.programme_curriculum_courseslot` | Curriculum course slots |
| CourseInstructor | `SELECT * FROM public.programme_curriculum_courseinstructor` | Teaching assignments |

### academic_information

| Table | SQL Query | Purpose |
|-------|-----------|---------|
| Student | `SELECT * FROM public.academic_information_student ORDER BY id_id ASC` | Student master (batch, category) |

### academic_procedures

| Table | SQL Query | Purpose |
|-------|-----------|---------|
| course_registration | `SELECT * FROM public.course_registration ORDER BY id ASC` | Course enrollments |
| InitialRegistration | `SELECT * FROM public."InitialRegistration" ORDER BY id ASC` | Initial course registration requests |
| FinalRegistration | `SELECT * FROM public."FinalRegistration" ORDER BY id ASC` | Verified final registrations |
| FeePayments | `SELECT * FROM public."FeePayments" ORDER BY id ASC` | Fee payment records |
| Dues | `SELECT * FROM public."Dues" ORDER BY id ASC` | Student dues |
| BranchChange | `SELECT * FROM public.academic_procedures_branchchange ORDER BY c_id ASC` | Branch transfer requests |

### online_cms (Production Table Only)

| Table | SQL Query | Purpose |
|-------|-----------|---------|
| Student_grades | `SELECT * FROM public.online_cms_student_grades` | Student course grades |

---

## Other Module Integration Guides

| # | Module | README File | Description |
|---|--------|-------------|-------------|
| 01 | LMS | [01_LMS_Integration_README.md](01_LMS_Integration_README.md) | Learning Management System |
| 02 | Award and Scholarship | [02_Award_Scholarship_Integration_README.md](02_Award_Scholarship_Integration_README.md) | Scholarships, awards, financial aid |
| 03 | Department | [03_Department_Integration_README.md](03_Department_Integration_README.md) | Department management |
| 04 | Other Academic Procedures | [04_Other_Academic_Procedure_Integration_README.md](04_Other_Academic_Procedure_Integration_README.md) | Additional academic workflows |
| 05 | File Tracking System | [05_File_Tracking_System_Integration_README.md](05_File_Tracking_System_Integration_README.md) | Document routing and tracking |
| 06 | RSPC | [06_RSPC_Integration_README.md](06_RSPC_Integration_README.md) | Research projects, publications |
| 07 | PS Management | [07_PS_Management_Integration_README.md](07_PS_Management_Integration_README.md) | Placement semester |
| 08 | HR/EIS | [08_HR_EIS_Integration_README.md](08_HR_EIS_Integration_README.md) | HR and employee information |
| 09 | PHC | [09_PHC_Integration_README.md](09_PHC_Integration_README.md) | Health center, medical services |
| 10 | Visitor Hostel | [10_Visitor_Hostel_Integration_README.md](10_Visitor_Hostel_Integration_README.md) | Guest accommodation |
| 11 | Hostel Management | [11_Hostel_Management_Integration_README.md](11_Hostel_Management_Integration_README.md) | Student hostel allocation |
| 12 | Mess Management | [12_Mess_Management_Integration_README.md](12_Mess_Management_Integration_README.md) | Mess operations and billing |
| 13 | Placement Cell | [13_Placement_Cell_Integration_README.md](13_Placement_Cell_Integration_README.md) | Campus recruitment |
| 14 | Gymkhana | [14_Gymkhana_Integration_README.md](14_Gymkhana_Integration_README.md) | Clubs, events, activities |
| 15 | Complaint Management | [15_Complaint_Management_Integration_README.md](15_Complaint_Management_Integration_README.md) | Grievance handling |
| 16 | IWD | [16_IWD_Integration_README.md](16_IWD_Integration_README.md) | Infrastructure works |
| 17 | System Administration | [17_System_Admin_Integration_README.md](17_System_Admin_Integration_README.md) | User and role management |
| 18 | Dashboards | [18_Dashboards_Integration_README.md](18_Dashboards_Integration_README.md) | Analytics and reporting |
| 19 | Backup & Archive | [19_Backup_Archive_Integration_README.md](19_Backup_Archive_Integration_README.md) | Backup, archival, disaster recovery |
| 20 | System Security | [20_System_Security_Integration_README.md](20_System_Security_Integration_README.md) | Security policies, 2FA, IP blocking |
| 21 | Patent Management | [21_Patent_Management_Integration_README.md](21_Patent_Management_Integration_README.md) | Intellectual property, patents |
| 22 | Visitor Management | [22_Visitor_Management_Integration_README.md](22_Visitor_Management_Integration_README.md) | Visitor registration, gate passes |
| 23 | Internal Audit | [23_Internal_Audit_Integration_README.md](23_Internal_Audit_Integration_README.md) | Audits, compliance, findings |

---

## Common Import Pattern

```python
from applications.programme_curriculum.models import (
    Programme, Discipline, Batch, Curriculum, 
    Semester, Course, CourseSlot, CourseInstructor
)
from applications.academic_information.models import Student
from applications.academic_procedures.models import (
    course_registration, FeePayments, Dues
)
from applications.online_cms.models import Student_grades
from applications.globals.models import ExtraInfo, Faculty, Staff
```

---

## Relationship Overview

```
globals.ExtraInfo (User extension)
    |
    ├── academic_information.Student (OneToOne)
    |       |
    |       └── batch_id --> programme_curriculum.Batch
    |                           |
    |                           ├── discipline --> Discipline
    |                           └── curriculum --> Curriculum
    |
    ├── globals.Faculty (OneToOne)
    |       |
    |       └── programme_curriculum.CourseInstructor
    |
    └── globals.Staff (OneToOne)

Student --> academic_procedures.course_registration --> Course, Semester
Student --> academic_procedures.FeePayments --> Semester
Student --> academic_procedures.Dues

online_cms.Student_grades --> roll_no (matches ExtraInfo.id)
```

---

## Sync Guidelines

1. Always use Foreign Keys to reference production tables
2. Never duplicate data - reference it
3. Use `select_related()` for optimized queries
4. Check `verified=True` for grades
5. Check `running_batch=True` for active batches
6. Check `working_course=True` and `latest_version=True` for courses
