# Academic Procedures Module - Integration Guide

**What this module is:** The Academic Procedures module handles all student academic transactions - course registration, fee payments, dues tracking, and branch change requests.

**Why it exists:** Academic operations are transactional. When a student registers for courses, pays fees, or applies for branch change, these are recorded as transactions. This module tracks the "what happened" of a student's academic journey - which courses they registered for, when they paid fees, what dues they owe.

**Why integration is needed:** Other modules need to check academic status. Placement needs to verify if a student has backlogs (from course_registration). Hostel/Mess need to check dues. Scholarships need to verify fee payment status. All these modules query this module's tables rather than maintaining separate records.

**Key tables:** course_registration, InitialRegistration, FinalRegistration, FeePayments, Dues, BranchChange

---

## Module Overview

The `academic_procedures` module handles student academic operations including course registration, fee payments, dues management, branch changes, and verification processes.

This module syncs with production and other modules can reference these tables for registration status and fee information.

---

## Tables (Sync with Production)

### Table: `course_registration`

Stores student course registrations per semester.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Primary Key |
| `working_year` | integer | Working year |
| `course_id_id` | integer | Course reference |
| `course_slot_id_id` | integer | Course slot reference |
| `semester_id_id` | integer | Semester reference |
| `student_id_id` | character varying | Student reference (roll number) |
| `registration_type` | character varying | Regular/Backlog/Audit/Improvement |
| `semester_type` | character varying | Odd/Even/Summer |
| `session` | character varying | Academic session (2024-25) |

```sql
SELECT * FROM public.course_registration ORDER BY id ASC
```

**Usage:** Enrollment verification, registered courses, backlog tracking.

---

### Table: `InitialRegistration`

Stores initial course registration before finalization.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Primary Key |
| `timestamp` | timestamp with time zone | Registration timestamp |
| `priority` | integer | Course priority |
| `course_id_id` | integer | Course reference |
| `course_slot_id_id` | integer | Course slot reference |
| `semester_id_id` | integer | Semester reference |
| `student_id_id` | character varying | Student reference (roll number) |
| `registration_type` | character varying | Registration type |
| `old_course_registration_id` | integer | Previous registration reference |

```sql
SELECT * FROM public."InitialRegistration" ORDER BY id ASC
```

**Usage:** Pre-registration tracking, priority allocation.

---

### Table: `FinalRegistration`

Stores finalized course registrations.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Primary Key |
| `verified` | boolean | Verification status |
| `course_id_id` | integer | Course reference |
| `course_slot_id_id` | integer | Course slot reference |
| `semester_id_id` | integer | Semester reference |
| `student_id_id` | character varying | Student reference (roll number) |
| `registration_type` | character varying | Registration type |
| `old_course_registration_id` | integer | Previous registration reference |

```sql
SELECT * FROM public."FinalRegistration" ORDER BY id ASC
```

**Usage:** Confirmed registrations, verified enrollments.

---

### Table: `FeePayments`

Stores student fee payment records.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Primary Key |
| `mode` | character varying | Payment mode |
| `transaction_id` | character varying | Payment transaction ID |
| `fee_receipt` | character varying | Receipt file path |
| `deposit_date` | date | Date of deposit |
| `utr_number` | character varying | UTR number |
| `fee_paid` | integer | Amount paid |
| `reason` | character varying | Reason for payment |
| `actual_fee` | integer | Total fee amount |
| `semester_id_id` | integer | Semester reference |
| `student_id_id` | character varying | Student reference (roll number) |

```sql
SELECT * FROM public."FeePayments" ORDER BY id ASC
```

**Usage:** Fee verification, payment status, financial clearance.

---

### Table: `Dues`

Stores student dues information.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Primary Key |
| `mess_due` | integer | Mess dues amount |
| `hostel_due` | integer | Hostel dues amount |
| `library_due` | integer | Library dues amount |
| `placement_cell_due` | integer | Placement cell dues |
| `academic_due` | integer | Academic dues amount |
| `student_id_id` | character varying | Student reference (roll number) |

```sql
SELECT * FROM public."Dues" ORDER BY id ASC
```

**Usage:** Due clearance, no-dues verification.

---

### Table: `academic_procedures_branchchange`

Stores branch change requests.

| Column | Type | Description |
|--------|------|-------------|
| `c_id` | integer | Primary Key |
| `applied_date` | date | Application date |
| `branches_id` | integer | Target branch reference |
| `user_id` | character varying | Student reference (roll number) |

```sql
SELECT * FROM public.academic_procedures_branchchange ORDER BY c_id ASC
```

**Usage:** Branch transfer tracking, status verification.

---

## How to Use in Your Module

### Import Statements

```python
from applications.academic_procedures.models import (
    course_registration,
    InitialRegistration,
    FinalRegistration,
    FeePayments,
    Dues,
    BranchChange
)
```

### Common Queries

```python
# Get all registered courses for a student in current semester
course_registration.objects.filter(
    student_id=student,
    semester_id=current_semester
)

# Get students registered for a course
course_registration.objects.filter(course_id=course)

# Check if student has paid fees
FeePayments.objects.filter(
    student_id=student,
    semester_id=semester
).exists()

# Get dues for a student
dues = Dues.objects.filter(student_id=student).first()

# Check if student has any pending dues
has_dues = False
if dues:
    has_dues = (dues.mess_due + dues.hostel_due + dues.library_due + 
                dues.placement_cell_due + dues.academic_due) > 0

# Get backlog courses
course_registration.objects.filter(
    student_id=student,
    registration_type='Backlog'
)
```

### Foreign Key Reference Pattern

```python
# In your module's model - if needed
class YourModel(models.Model):
    # Usually you reference Student directly, not course_registration
    student = models.ForeignKey(
        'academic_information.Student',
        on_delete=models.CASCADE
    )
```

---

## Common Integration Patterns

### Verify Student Registration

```python
def is_student_registered(student, course, semester):
    """Check if student is registered for a course"""
    return course_registration.objects.filter(
        student_id=student,
        course_id=course,
        semester_id=semester
    ).exists()
```

### Get Registered Students for a Course

```python
def get_course_students(course, semester):
    """Get all students registered in a course"""
    registrations = course_registration.objects.filter(
        course_id=course,
        semester_id=semester
    ).select_related('student_id', 'student_id__id')
    
    return [reg.student_id for reg in registrations]
```

### Check Fee Clearance

```python
def is_fee_cleared(student, semester):
    """Check if student has cleared fees"""
    payment = FeePayments.objects.filter(
        student_id=student,
        semester_id=semester
    ).first()
    
    if payment:
        return payment.fee_paid >= payment.actual_fee
    return False
```

### Check No Dues

```python
def has_no_dues(student):
    """Check if student has no pending dues"""
    dues = Dues.objects.filter(student_id=student).first()
    
    if not dues:
        return True  # No dues record means no dues
    
    total_dues = (
        dues.mess_due + 
        dues.hostel_due + 
        dues.library_due + 
        dues.placement_cell_due + 
        dues.academic_due
    )
    
    return total_dues == 0

def get_total_dues(student):
    """Get total dues amount for a student"""
    dues = Dues.objects.filter(student_id=student).first()
    
    if not dues:
        return 0
    
    return (
        dues.mess_due + 
        dues.hostel_due + 
        dues.library_due + 
        dues.placement_cell_due + 
        dues.academic_due
    )
```

### Count Backlogs

```python
def count_backlogs(student):
    """Count number of backlog courses for a student"""
    return course_registration.objects.filter(
        student_id=student,
        registration_type='Backlog'
    ).count()
```

---

## Relationship Diagram

```
Student (academic_information)
    |
    ├── course_registration
    |       ├── semester_id --> Semester (programme_curriculum)
    |       ├── course_id --> Course (programme_curriculum)
    |       └── course_slot_id --> CourseSlot (programme_curriculum)
    |
    ├── FeePayments
    |       └── semester_id --> Semester
    |
    ├── Dues
    |
    └── BranchChange
```

---

## Sync Points for Other Modules

| Your Module Needs | Use This Table/Field |
|-------------------|---------------------|
| Course enrollment status | course_registration |
| Registered students | course_registration.student_id |
| Fee payment status | FeePayments |
| Dues clearance | Dues |
| Backlog count | course_registration (registration_type='Backlog') |
| Branch change status | BranchChange |
