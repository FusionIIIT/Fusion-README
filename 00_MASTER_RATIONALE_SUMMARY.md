# Fusion ERP - Master Rationale Summary

## Overview

This document provides a consolidated summary of all module integration guides for Fusion ERP. Each section explains what the module does, why it exists, why integration is needed, and its key dependencies.

---

## Table of Contents

1. [Production Sync Modules](#production-sync-modules)
   - [Programme Curriculum](#00-programme-curriculum)
   - [Academic Information](#00-academic-information)
   - [Academic Procedures](#00-academic-procedures)
   - [Online CMS](#00-online-cms)
2. [Other Modules](#other-modules)
   - [01 - LMS](#01-lms-learning-management-system)
   - [02 - Award and Scholarship](#02-award-and-scholarship)
   - [03 - Department](#03-department)
   - [04 - Other Academic Procedures](#04-other-academic-procedures)
   - [05 - File Tracking System](#05-file-tracking-system)
   - [06 - RSPC](#06-rspc-research-sponsored-projects-and-consultancy)
   - [07 - P&S Management](#07-ps-purchase-and-store-management)
   - [08 - HR/EIS](#08-hreis-employee-information-system)
   - [09 - PHC](#09-phc-primary-health-center)
   - [10 - Visitor Hostel](#10-visitor-hostel)
   - [11 - Hostel Management](#11-hostel-management)
   - [12 - Mess Management](#12-mess-management)
   - [13 - Placement Cell](#13-placement-cell)
   - [14 - Gymkhana](#14-gymkhana)
   - [15 - Complaint Management](#15-complaint-management)
   - [16 - IWD](#16-iwd-institute-works-department)
   - [17 - System Administration](#17-system-administration)
   - [18 - Dashboards](#18-dashboards)
   - [19 - Backup & Archive](#19-backup--archive)
   - [20 - System Security](#20-system-security)
   - [21 - Patent Management](#21-patent-management)
   - [22 - Visitor Management](#22-visitor-management)
   - [23 - Internal Audit](#23-internal-audit)

---

# Production Sync Modules

These modules contain the core production tables that other modules sync with and reference.

---

## 00. Programme Curriculum

| Aspect | Description |
|--------|-------------|
| **What it is** | The foundational academic structure of Fusion ERP. Defines the academic hierarchy - programmes (B.Tech, M.Tech, PhD), disciplines/branches (CSE, ECE, ME), active batches, and available courses. |
| **Why it exists** | Every academic operation needs to know the academic structure. Course registration needs course info. Transcripts need curriculum info. Single source of truth for all academic structure data. |
| **Why integration needed** | All other modules (Placement, Hostel, Mess, Scholarships) reference this data via Foreign Keys. Ensures consistency - if a course name changes, it changes everywhere. |
| **Key tables** | Programme, Discipline, Batch, Course, Semester, CourseSlot, CourseInstructor |
| **Status** | ✅ Production - Sync Required |

---

## 00. Academic Information

| Aspect | Description |
|--------|-------------|
| **What it is** | Stores the master student data - every enrolled student's basic academic information including roll number, batch, programme, category, and hostel details. |
| **Why it exists** | The Student table is the central identity for all student-related operations. Every module dealing with students (Placement, Hostel, Mess, Library, Scholarships) needs to identify and reference students. |
| **Why integration needed** | Rather than each module storing duplicate student information, they all link to this module's Student table. When a student's batch changes, it updates in one place and all modules see the updated data. |
| **Key tables** | Student (linked to ExtraInfo for user details) |
| **Status** | ✅ Production - Sync Required |

---

## 00. Academic Procedures

| Aspect | Description |
|--------|-------------|
| **What it is** | Handles all student academic transactions - course registration, fee payments, dues tracking, and branch change requests. |
| **Why it exists** | Academic operations are transactional. This module tracks "what happened" in a student's academic journey - courses registered, fees paid, dues owed. |
| **Why integration needed** | Other modules check academic status. Placement verifies backlogs (from course_registration). Hostel/Mess check dues. Scholarships verify fee payment status. All query this module. |
| **Key tables** | course_registration, InitialRegistration, FinalRegistration, FeePayments, Dues, BranchChange |
| **Status** | ✅ Production - Sync Required |

---

## 00. Online CMS

| Aspect | Description |
|--------|-------------|
| **What it is** | Designed for online learning features. In production, only the Student_grades table is actively used - stores all student course grades. |
| **Why it exists** | Student grades are fundamental to academic records. This table is the source of truth for student academic performance - transcripts, eligibility checks, backlog verification. |
| **Why integration needed** | Multiple modules need grade data. Placement checks backlogs (grade='F'). Scholarships verify academic performance. Dashboards display grade summaries. All query Student_grades. |
| **Key tables** | Student_grades (only production table - other tables not used) |
| **Status** | ✅ Production - Sync Required |

---

# Other Modules

These modules need to be built or enhanced, integrating with production data.

---

## 01. LMS (Learning Management System)

| Aspect | Description |
|--------|-------------|
| **What it is** | Will handle online course delivery - lecture materials, resources, attendance tracking, quizzes, assignments, and discussions. |
| **Why it exists** | No integrated digital learning platform currently. Faculty share materials via email or external drives. Assignments submitted physically. Quizzes on paper. |
| **Why integration needed** | Must know which student is enrolled in which course (from course_registration), who teaches which course (from CourseInstructor), student identity (from Student). |
| **Key dependencies** | Student, course_registration, Courses, CourseInstructor, ExtraInfo |
| **Status** | 🔨 To Be Built |

---

## 02. Award and Scholarship

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage all financial aid and recognition programs - merit scholarships, need-based aid, government scholarships, sports awards, academic excellence recognitions. |
| **Why it exists** | Students receive various scholarships but eligibility is verified manually using spreadsheets. Module will automate the scholarship lifecycle - applications, eligibility checks, approvals, disbursement tracking. |
| **Why integration needed** | Eligibility depends on data from core modules. Need to verify batch/programme (from programme_curriculum), check backlogs (from Student_grades), verify fee payment (from FeePayments), check dues (from Dues). |
| **Key dependencies** | Student, Batch, Programme, Student_grades, FeePayments, Dues |
| **Status** | 🔨 To Be Built |

---

## 03. Department

| Aspect | Description |
|--------|-------------|
| **What it is** | Will handle department-level operations - managing faculty, announcements, research coordination, curriculum oversight, department-specific student interactions. |
| **Why it exists** | Each department (CSE, ECE, ME) has administrative needs but HODs lack a unified view. Will provide department-scoped views - students, faculty workload, course offerings, research activities. |
| **Why integration needed** | Department data comes from core modules. Students belong to batches which belong to disciplines. Faculty teach courses. All data exists in programme_curriculum and academic_information. |
| **Key dependencies** | Discipline, Batch, Student, Course, CourseInstructor, Faculty |
| **Status** | 🔨 To Be Built |

---

## 04. Other Academic Procedures

| Aspect | Description |
|--------|-------------|
| **What it is** | Will handle miscellaneous academic requests - bonafide certificates, thesis topic approval, assistantship claims, branch change applications, no-dues certificates, special permissions. |
| **Why it exists** | Beyond course registration, students need various academic services. Currently paper-based. Module will automate workflows - digital applications, automatic verification, faster processing. |
| **Why integration needed** | Every request requires student verification. Branch change needs batch info. Thesis approval needs supervisor (Faculty) info. Assistantship verifies student status. No-dues checks Dues table. |
| **Key dependencies** | Student, Batch, Faculty, Dues, BranchChange |
| **Status** | 🔨 To Be Built |

---

## 05. File Tracking System

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage document workflow across the institute - tracking files moving between offices, managing approvals, maintaining audit trails for administrative documents. |
| **Why it exists** | Files (leave applications, purchase requests, student petitions) move through multiple offices. File location is often unknown causing delays. Will provide real-time file tracking. |
| **Why integration needed** | Files routed to departments (Discipline), sent by employees (ExtraInfo/Faculty/Staff), may concern students (Student). Needs organizational structure to route files correctly. |
| **Key dependencies** | Discipline, ExtraInfo, Faculty, Staff, DepartmentInfo |
| **Status** | 🔨 To Be Built |

---

## 06. RSPC (Research, Sponsored Projects and Consultancy)

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage research activities - sponsored projects (funded research), consultancy services, patent filings, publications, and research funding applications. |
| **Why it exists** | Faculty and students conduct research that needs tracking. Sponsored projects managed in spreadsheets with budgets, timelines tracked manually. Will provide unified project lifecycle management. |
| **Why integration needed** | Research conducted by Faculty (from globals), may involve PhD students (from Student), belongs to disciplines/departments. All identity and organizational data comes from core modules. |
| **Key dependencies** | Faculty, Student, Discipline, ExtraInfo |
| **Status** | 🔨 To Be Built |

---

## 07. P&S (Purchase and Store Management)

| Aspect | Description |
|--------|-------------|
| **What it is** | Will handle procurement - purchase requests (indents), vendor management, purchase orders, goods receipt, inventory tracking, and store operations. |
| **Why it exists** | Indents are paper-based, vendor records scattered. Will digitize the entire procurement workflow - indent to goods receipt - with proper approvals and inventory tracking. |
| **Why integration needed** | Purchase requests come from departments (Discipline) and are raised by employees (ExtraInfo/Faculty/Staff). Approvals route through organizational hierarchy. |
| **Key dependencies** | Discipline, ExtraInfo, Faculty, Staff, DepartmentInfo |
| **Status** | 🔨 To Be Built |

---

## 08. HR/EIS (Employee Information System)

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage employee lifecycle - joining to retirement. Tracks service records, qualifications, performance appraisals, training, promotions, leave. |
| **Why it exists** | Institute has faculty, staff, and administrators. HR processes are paper-based. Will provide digital employee management with self-service for leave applications and service record viewing. |
| **Why integration needed** | Employees exist in globals (ExtraInfo, Faculty, Staff). They belong to departments (Discipline). Faculty teach courses (CourseInstructor). HR/EIS extends this with service-specific information. |
| **Key dependencies** | ExtraInfo, Faculty, Staff, Discipline, CourseInstructor |
| **Status** | 🔨 To Be Built |

---

## 09. PHC (Primary Health Center)

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage the institute's health center - patient records, consultations, prescriptions, medical history, ambulance services, and health insurance claims. |
| **Why it exists** | Students and staff need healthcare. Medical records are paper-based, making history tracking difficult. Will digitize health center operations - appointments, prescriptions, medical history. |
| **Why integration needed** | Patients are either students (from Student table) or employees (from ExtraInfo/Faculty/Staff). PHC needs to identify patients and access their basic information. |
| **Key dependencies** | Student, ExtraInfo, Faculty, Staff |
| **Status** | 🔨 To Be Built |

---

## 10. Visitor Hostel

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage the institute's guest house - room bookings, visitor registration, billing, and accommodation for official guests, conference attendees, visiting faculty. |
| **Why it exists** | Booking is via phone/email, room availability checked manually. Will provide online room booking, automated availability checks, and digital billing. |
| **Why integration needed** | Bookings made by institute members (ExtraInfo/Faculty/Staff) for guests. Host's identity and department need verification. Some bookings may be charged to project accounts. |
| **Key dependencies** | ExtraInfo, Faculty, Staff, Discipline |
| **Status** | 🔨 To Be Built |

---

## 11. Hostel Management

| Aspect | Description |
|--------|-------------|
| **What it is** | Will handle student accommodation - room allocation, hostel transfers, leave requests, visitor entry, complaints, and hostel administration. |
| **Why it exists** | Room allocation is manual, leave requests on paper. Will provide automated room allocation based on batch/programme/gender, digital leave requests, and warden dashboards. |
| **Why integration needed** | Hostel residents are students (from Student table). Room allocation considers batch and programme. Student's current hostel info is in academic_information. |
| **Key dependencies** | Student, Batch, ExtraInfo, Dues (for hostel dues) |
| **Status** | 🔨 To Be Built |

---

## 12. Mess Management

| Aspect | Description |
|--------|-------------|
| **What it is** | Will handle dining operations - menu planning, student mess registration, meal tracking, billing, rebates for missed meals, feedback collection, and mess worker management. |
| **Why it exists** | Mess registration is manual, rebate requests on paper. Will digitize mess operations - online registration, automated billing, digital rebate workflows. |
| **Why integration needed** | Mess members are students (from Student table). Mess dues sync with Dues table in academic_procedures. Rebates during hostel leave link to Hostel module. |
| **Key dependencies** | Student, Batch, ExtraInfo, Dues (mess_due field) |
| **Status** | 🔨 To Be Built |

---

## 13. Placement Cell

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage campus recruitment - company registrations, job postings, student placement profiles, application tracking, interview scheduling, offer management, and statistics. |
| **Why it exists** | Process is fragmented - companies register via email, students submit profiles on paper, eligibility checked manually. Will digitize the entire placement process with automated eligibility checks. |
| **Why integration needed** | Must verify student identity (from Student), check batch/discipline (from programme_curriculum), verify backlog status (from Student_grades), get contact details (from ExtraInfo). |
| **Key dependencies** | Student, Batch, Discipline, Programme, Student_grades, ExtraInfo |
| **Status** | 🔨 To Be Built |

---

## 14. Gymkhana

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage student extracurricular life - clubs (technical, cultural, sports), societies, events, budgets, inventory, elections for student council, activity registrations. |
| **Why it exists** | Club management is manual - member lists on paper, budgets in spreadsheets, event registrations via forms. Will provide digital club management, automated elections, centralized event coordination. |
| **Why integration needed** | Club members are students (from Student table). Faculty advisors link to Faculty table. Election eligibility may require academic standing checks. |
| **Key dependencies** | Student, Batch, ExtraInfo, Faculty |
| **Status** | 🔨 To Be Built |

---

## 15. Complaint Management

| Aspect | Description |
|--------|-------------|
| **What it is** | Will handle grievance redressal - complaint registration, categorization (academic/hostel/mess/infrastructure), assignment to handlers, tracking, resolution, feedback. |
| **Why it exists** | Complaints are verbal or via scattered emails with no tracking. Will provide structured complaint registration, automatic routing, and resolution tracking. |
| **Why integration needed** | Complainants are students (Student) or employees (ExtraInfo/Staff). Complaints routed to departments (DepartmentInfo) or specific authorities. |
| **Key dependencies** | Student, ExtraInfo, Faculty, Staff, Batch, Discipline |
| **Status** | 🔨 To Be Built |

---

## 16. IWD (Institute Works Department)

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage infrastructure - construction projects, maintenance requests, electrical/civil works, contractor management, work orders, building/asset management. |
| **Why it exists** | Work requests are manual, tracking on paper, contractor management fragmented. Will provide digital work order management, project tracking, maintenance request workflows. |
| **Why integration needed** | Work requests come from departments (DepartmentInfo) and employees (ExtraInfo/Faculty/Staff). Approvals flow through organizational hierarchy. |
| **Key dependencies** | ExtraInfo, Faculty, Staff, Discipline, DepartmentInfo |
| **Status** | 🔨 To Be Built |

---

## 17. System Administration

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage user accounts, role assignments, permissions, system configurations, access controls, audit logs, and overall system settings. |
| **Why it exists** | User management is scattered. Will provide centralized account creation, role assignment (student/faculty/admin), permission management, and system configuration. |
| **Why integration needed** | Controls ALL other modules. Assigns roles based on user type (Student/Faculty/Staff from globals). Configures module-specific settings. Needs access to all identity tables. |
| **Key dependencies** | ExtraInfo, Student, Faculty, Staff, all module tables (for permissions) |
| **Status** | 🔨 To Be Built |

---

## 18. Dashboards

| Aspect | Description |
|--------|-------------|
| **What it is** | Will provide role-based summary views - students see courses and grades, faculty see advisees and classes, admins see analytics and metrics. |
| **Why it exists** | Users need a personalized landing page. A student wants courses, dues, notifications at a glance. A faculty wants classes and pending approvals. No unified dashboard currently. |
| **Why integration needed** | Dashboards READ from ALL production modules. Student dashboards need course_registration, Student_grades, Dues. Faculty dashboards need thesis supervision, department info. Ultimate consumer of all tables. |
| **Key dependencies** | ALL production tables (Student, course_registration, Student_grades, Dues, ExtraInfo, etc.) |
| **Status** | 🔨 To Be Built |

---

## 19. Backup & Archive

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage data backups, archival of old records, retention policies, and disaster recovery procedures. |
| **Why it exists** | No systematic backup strategy currently. Graduated students' records need archival. Active data needs regular backups. Will ensure data persistence and enable disaster recovery. |
| **Why integration needed** | Backups touch ALL tables. Archival policies may differ by data type. Module needs to understand schema structure across all production modules to backup and restore correctly. |
| **Key dependencies** | ALL tables (for backup/restore), system configuration |
| **Status** | 🔨 To Be Built |

---

## 20. System Security

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage authentication, authorization, access control lists, security policies, threat monitoring, and compliance. |
| **Why it exists** | ERP systems contain sensitive data - grades, finances, personal info. Security is basic currently. Will enforce security policies, implement role-based access, detect unauthorized access. |
| **Why integration needed** | Security spans ALL modules. Access control checks user roles (from ExtraInfo). Audit logs track who accessed what data. Monitors all API endpoints and database access. |
| **Key dependencies** | Django User model, ExtraInfo, ALL modules (for access control) |
| **Status** | 🔨 To Be Built |

---

## 21. Patent Management

| Aspect | Description |
|--------|-------------|
| **What it is** | Will track intellectual property - invention disclosures, patent applications, filing status, inventor details, and technology transfer. |
| **Why it exists** | Research institutes generate IP but patent tracking is manual. Faculty disclose inventions on paper, filing status in spreadsheets. Will provide digital invention disclosure and IP portfolio management. |
| **Why integration needed** | Inventors are faculty (Faculty) or students (Student) identified by ExtraInfo. Patents filed from departments (DepartmentInfo). Needs identity and organizational data. |
| **Key dependencies** | ExtraInfo, Faculty, Student, DepartmentInfo |
| **Status** | 🔨 To Be Built |

---

## 22. Visitor Management

| Aspect | Description |
|--------|-------------|
| **What it is** | Will handle campus visitor registration, gate passes, entry/exit tracking, host verification, and visitor logs. |
| **Why it exists** | Visitor tracking is manual - visitors sign a register, hosts called for verification. Will provide digital visitor registration, automated host notification, and real-time tracking. |
| **Why integration needed** | Hosts are faculty/staff (from ExtraInfo/Faculty/Staff). Visitors may visit specific departments (DepartmentInfo). Must verify host identity and organizational affiliation. |
| **Key dependencies** | ExtraInfo, Faculty, Staff, DepartmentInfo |
| **Status** | 🔨 To Be Built |

---

## 23. Internal Audit

| Aspect | Description |
|--------|-------------|
| **What it is** | Will manage compliance audits, process reviews, audit scheduling, findings tracking, and corrective action monitoring. |
| **Why it exists** | Audits conducted manually with findings tracked in documents. Will provide structured audit scheduling, digital findings management, and corrective action tracking. |
| **Why integration needed** | Audits examine data from ALL modules - finance records, academic processes, inventory. Auditors and auditees are employees (ExtraInfo). Department audits link to DepartmentInfo. |
| **Key dependencies** | ExtraInfo, DepartmentInfo, ALL modules (for audit data) |
| **Status** | 🔨 To Be Built |

---

# Quick Reference: Module Dependencies

| Module | Key Dependencies |
|--------|------------------|
| Programme Curriculum | - (Core module) |
| Academic Information | ExtraInfo, Batch |
| Academic Procedures | Student, Course, Semester |
| Online CMS | Course, ExtraInfo |
| LMS | Student, course_registration, CourseInstructor |
| Award & Scholarship | Student, Batch, Student_grades, FeePayments, Dues |
| Department | Discipline, Batch, Student, Course, Faculty |
| Other Academic Procedures | Student, Batch, Faculty, Dues |
| File Tracking | Discipline, ExtraInfo, Faculty, Staff |
| RSPC | Faculty, Student, Discipline |
| P&S Management | Discipline, ExtraInfo, Faculty, Staff |
| HR/EIS | ExtraInfo, Faculty, Staff, CourseInstructor |
| PHC | Student, ExtraInfo, Faculty, Staff |
| Visitor Hostel | ExtraInfo, Faculty, Staff |
| Hostel Management | Student, Batch, ExtraInfo, Dues |
| Mess Management | Student, Batch, ExtraInfo, Dues |
| Placement Cell | Student, Batch, Discipline, Student_grades |
| Gymkhana | Student, Batch, ExtraInfo, Faculty |
| Complaint Management | Student, ExtraInfo, Faculty, Staff |
| IWD | ExtraInfo, Faculty, Staff, DepartmentInfo |
| System Admin | ExtraInfo, Student, Faculty, Staff, ALL modules |
| Dashboards | ALL production tables |
| Backup & Archive | ALL tables |
| System Security | User, ExtraInfo, ALL modules |
| Patent Management | ExtraInfo, Faculty, Student, DepartmentInfo |
| Visitor Management | ExtraInfo, Faculty, Staff, DepartmentInfo |
| Internal Audit | ExtraInfo, DepartmentInfo, ALL modules |

---

# Integration Architecture Summary

```
                    ┌─────────────────────────────────────────┐
                    │           CORE PRODUCTION MODULES        │
                    │  (Single Source of Truth - Read Only)   │
                    └─────────────────────────────────────────┘
                                        │
         ┌──────────────────────────────┼──────────────────────────────┐
         │                              │                              │
         ▼                              ▼                              ▼
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│   programme     │          │    academic     │          │   online_cms    │
│   _curriculum   │          │  _information   │          │ (Student_grades)│
├─────────────────┤          ├─────────────────┤          └─────────────────┘
│ • Programme     │          │ • Student       │                  │
│ • Discipline    │◄─────────┤ (links to       │◄─────────────────┘
│ • Batch         │          │  Batch, Extra   │
│ • Course        │          │  Info)          │
│ • Semester      │          └─────────────────┘
│ • CourseSlot    │                  │
│ • CourseInst.   │                  │
└─────────────────┘                  │
         │                           │
         └───────────────┬───────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │ academic_procedures │
              ├─────────────────────┤
              │ • course_registration│
              │ • FeePayments       │
              │ • Dues              │
              │ • BranchChange      │
              └─────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────────────────┐
         │          ALL OTHER MODULES                │
         │  (Read from core, write to their own)     │
         ├───────────────────────────────────────────┤
         │ LMS, Placement, Hostel, Mess, PHC,        │
         │ Gymkhana, Complaints, Awards, Department, │
         │ RSPC, P&S, HR, IWD, Visitor, FTS,         │
         │ Dashboards, Admin, Security, Audit, etc.  │
         └───────────────────────────────────────────┘
```

---

# Common Import Pattern

```python
# Core Production Modules
from applications.programme_curriculum.models import (
    Programme, Discipline, Batch, Curriculum, 
    Semester, Course, CourseSlot, CourseInstructor
)
from applications.academic_information.models import Student
from applications.online_cms.models import Student_grades
from applications.academic_procedures.models import (
    course_registration, FeePayments, Dues, BranchChange
)

# Globals (Identity)
from applications.globals.models import (
    ExtraInfo, Faculty, Staff, DepartmentInfo, 
    Designation, HoldsDesignation
)
```

---

*Last Updated: January 2026*
