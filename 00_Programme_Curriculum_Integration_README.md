# Programme Curriculum Module - Integration Guide

**What this module is:** The Programme Curriculum module is the foundational academic structure of Fusion ERP. It defines the academic hierarchy - what programmes exist (B.Tech, M.Tech, PhD), what disciplines/branches are offered (CSE, ECE, ME), which batches are active, and what courses are available.

**Why it exists:** Every academic operation in the ERP needs to know the academic structure. When a student registers for courses, the system needs to know what courses exist. When generating a transcript, it needs curriculum information. This module is the single source of truth for all academic structure data.

**Why integration is needed:** All other modules (Placement, Hostel, Mess, Scholarships, etc.) need to reference this data. Instead of each module maintaining its own copy of programmes and courses, they all reference this module's tables via Foreign Keys. This ensures consistency - if a course name changes, it changes everywhere.

**Key tables:** Programme, Discipline, Batch, Course, Semester, CourseSlot, CourseInstructor

---

## Module Overview

The `programme_curriculum` module provides foundational academic structure data. This module syncs with production and other modules can reference these tables.

This module manages programmes, disciplines, batches, courses, curriculum structure, and course assignments.

---

## Tables (Sync with Production)

These tables are available for other modules to reference via Foreign Keys.

### Table: `programme_curriculum_programme`

Stores degree programmes offered by the institute.

| Column | Type | Description |
|--------|------|-------------|
| `id` | AutoField | Primary Key |
| `category` | CharField(20) | UG/PG/PHD |
| `name` | CharField(70) | Programme name (B.Tech, M.Tech, PhD) |
| `programme_begin_year` | PositiveIntegerField | Year programme started |

```sql
SELECT * FROM public.programme_curriculum_programme ORDER BY id ASC
```

**Usage:** Reference for programme-level filtering and eligibility checks.

---

### Table: `programme_curriculum_discipline`

Stores academic branches/departments.

| Column | Type | Description |
|--------|------|-------------|
| `id` | AutoField | Primary Key |
| `name` | CharField(100) | Full discipline name |
| `acronym` | CharField(10) | Short code (CSE, ECE, ME) |
| `programmes` | ManyToManyField | Linked programmes |

```sql
SELECT * FROM public.programme_curriculum_discipline ORDER BY id ASC
```

**Usage:** Department/branch filtering, discipline-specific operations.

---

### Table: `programme_curriculum_batch`

Stores student batches per discipline.

| Column | Type | Description |
|--------|------|-------------|
| `id` | AutoField | Primary Key |
| `name` | CharField(50) | Batch name |
| `discipline` | ForeignKey(Discipline) | Discipline reference |
| `year` | PositiveIntegerField | Batch year (2020, 2021, etc.) |
| `curriculum` | ForeignKey(Curriculum) | Assigned curriculum |
| `running_batch` | BooleanField | Is batch currently active |

```sql
SELECT * FROM public.programme_curriculum_batch ORDER BY id ASC
```

**Usage:** Student grouping, batch-wise operations, passing out year.

---

### Table: `programme_curriculum_curriculum`

Stores curriculum versions.

| Column | Type | Description |
|--------|------|-------------|
| `id` | AutoField | Primary Key |
| `name` | CharField(100) | Curriculum name |
| `version` | PositiveIntegerField | Version number |
| `programme` | ForeignKey(Programme) | Programme reference |
| `working_curriculum` | BooleanField | Is active curriculum |
| `no_of_semester` | PositiveIntegerField | Total semesters |
| `min_credit` | PositiveIntegerField | Minimum credits required |

```sql
SELECT * FROM public.programme_curriculum_curriculum ORDER BY id ASC
```

**Usage:** Curriculum mapping, credit requirements, academic structure.

---

### Table: `programme_curriculum_semester`

Stores semester information per curriculum.

| Column | Type | Description |
|--------|------|-------------|
| `id` | AutoField | Primary Key |
| `curriculum` | ForeignKey(Curriculum) | Curriculum reference |
| `semester_no` | PositiveIntegerField | Semester number (1-8) |
| `start_semester` | DateField | Semester start date |
| `end_semester` | DateField | Semester end date |
| `instigate_open` | BooleanField | Registration open |

```sql
SELECT * FROM public.programme_curriculum_semester ORDER BY id ASC
```

**Usage:** Semester context, date validation, registration windows.

---

### Table: `programme_curriculum_course`

Stores all courses offered.

| Column | Type | Description |
|--------|------|-------------|
| `id` | AutoField | Primary Key |
| `code` | CharField(10) | Course code (CS101) |
| `name` | CharField(100) | Course name |
| `version` | DecimalField(4,1) | Course version |
| `credit` | PositiveIntegerField | Credit hours |
| `lecture_hours` | PositiveIntegerField | Lecture hours/week |
| `tutorial_hours` | PositiveIntegerField | Tutorial hours/week |
| `pratical_hours` | PositiveIntegerField | Lab hours/week |
| `syllabus` | TextField | Course syllabus |
| `working_course` | BooleanField | Is course active |
| `latest_version` | BooleanField | Is latest version |

```sql
SELECT * FROM public.programme_curriculum_course ORDER BY id ASC
```

**Usage:** Course references, credit calculation, prerequisite checks.

---

### Table: `programme_curriculum_courseslot`

Maps courses to curriculum slots.

| Column | Type | Description |
|--------|------|-------------|
| `id` | AutoField | Primary Key |
| `semester` | ForeignKey(Semester) | Semester reference |
| `name` | CharField(100) | Slot name |
| `type` | CharField(20) | Core/Elective/Open |
| `courses` | ManyToManyField(Course) | Courses in this slot |
| `min_registration_limit` | PositiveIntegerField | Min students |
| `max_registration_limit` | PositiveIntegerField | Max students |

```sql
SELECT * FROM public.programme_curriculum_courseslot ORDER BY id ASC
```

**Usage:** Course-semester mapping, slot-based registration.

---

### Table: `programme_curriculum_courseinstructor`

Stores course-instructor assignments.

| Column | Type | Description |
|--------|------|-------------|
| `id` | AutoField | Primary Key |
| `course_id` | ForeignKey(Course) | Course reference |
| `instructor_id` | ForeignKey(ExtraInfo) | Faculty ID |
| `batch_id` | ForeignKey(Batch) | Batch reference |
| `year` | IntegerField | Academic year |
| `semester_type` | CharField(20) | Odd/Even/Summer |

```sql
SELECT * FROM public.programme_curriculum_courseinstructor ORDER BY id ASC
```

**Usage:** Teaching assignments, instructor lookup.

---

## How to Use in Your Module

### Import Statements

```python
from applications.programme_curriculum.models import (
    Programme,
    Discipline,
    Batch,
    Curriculum,
    Semester,
    Course,
    CourseSlot,
    CourseInstructor
)
```

### Common Queries

```python
# Get all active programmes
Programme.objects.all()

# Get all disciplines under UG
disciplines = Discipline.objects.filter(programmes__category='UG')

# Get current running batches
Batch.objects.filter(running_batch=True)

# Get active curriculum for a programme
Curriculum.objects.filter(programme=programme, working_curriculum=True)

# Get course by code
Course.objects.filter(code='CS101', working_course=True, latest_version=True)

# Get courses for a semester
CourseSlot.objects.filter(semester=semester).prefetch_related('courses')

# Get instructor for a course
CourseInstructor.objects.filter(course_id=course, batch_id=batch)
```

### Foreign Key Reference Pattern

```python
# In your module's model
class YourModel(models.Model):
    batch = models.ForeignKey(
        'programme_curriculum.Batch',
        on_delete=models.CASCADE
    )
    course = models.ForeignKey(
        'programme_curriculum.Course',
        on_delete=models.CASCADE
    )
```

---

## Relationship Diagram

```
Programme
    |
    └── Discipline (ManyToMany)
            |
            └── Batch
                    |
                    └── Curriculum
                            |
                            └── Semester
                                    |
                                    └── CourseSlot
                                            |
                                            └── Course (ManyToMany)
```

---

## Sync Points for Other Modules

| Your Module Needs | Use This Table |
|-------------------|----------------|
| Programme type (UG/PG/PhD) | Programme |
| Branch/Department | Discipline |
| Student batch/year | Batch |
| Course information | Course |
| Semester dates | Semester |
| Teaching faculty | CourseInstructor |
| Curriculum structure | Curriculum, CourseSlot |
