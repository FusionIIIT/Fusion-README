# Online CMS Module - Integration Guide

**What this module is:** The Online CMS (Course Management System) module was designed for online learning features. However, in production, only the Student_grades table is actively used - it stores all student course grades.

**Why it exists:** Student grades are fundamental to academic records. Every course a student takes has a grade. This table is the source of truth for student academic performance - used for transcripts, eligibility checks, backlog verification, and academic standing.

**Why integration is needed:** Multiple modules need grade data. Placement checks if students have backlogs (grade='F'). Scholarships verify academic performance. Dashboards display grade summaries. All these modules query Student_grades rather than maintaining separate grade records.

**Key tables:** Student_grades (only production table - other online_cms tables are not used)

---

## Module Overview

The `online_cms` module contains the Student Grades table which is the only production-relevant table from this module.

Note: Other tables in online_cms (attendance, quizzes, assignments, forums, etc.) are NOT used in production. Only the `Student_grades` table is actively used.

---

## Tables (Sync with Production)

### Table: `online_cms_student_grades` (PRODUCTION TABLE)

This is the PRIMARY table for storing student course grades in Fusion ERP.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Primary Key |
| `semester` | integer | Semester number (1-8) |
| `year` | integer | Calendar year |
| `roll_no` | text | Student roll number |
| `grade` | text | Grade (A, A-, B+, B, B-, C+, C, C-, D, F) |
| `batch` | integer | Batch year |
| `course_id_id` | integer | Course reference |
| `reSubmit` | boolean | Resubmission flag |
| `remarks` | character varying | Remarks |
| `verified` | boolean | Grade verification status |
| `academic_year` | character varying | Academic year (2024-25) |
| `semester_type` | character varying | Odd/Even/Summer |

```sql
SELECT * FROM public.online_cms_student_grades ORDER BY id ASC
```

**Usage:** Grade retrieval, SGPA calculation, academic performance, eligibility checks.

---

---

## How to Use in Your Module

### Import Statements

```python
from applications.online_cms.models import Student_grades
```

### Common Queries

```python
# Get all grades for a student
Student_grades.objects.filter(roll_no='21BCS001').order_by('year', 'semester')

# Get verified grades only
Student_grades.objects.filter(roll_no='21BCS001', verified=True)

# Get grades for a specific semester
Student_grades.objects.filter(
    roll_no='21BCS001',
    semester=3,
    academic_year='2022-23'
)

# Get all grades for a course
Student_grades.objects.filter(
    course_id=course_id,
    semester=semester,
    year=year
)

# Get grades by batch
Student_grades.objects.filter(batch=2021, verified=True)

# Count failed courses (backlogs)
Student_grades.objects.filter(
    roll_no='21BCS001',
    grade='F',
    verified=True
).count()
```

---

## Common Integration Patterns

### Get Student Grades

```python
def get_student_grades(roll_no):
    """Get all grades for a student by roll number"""
    return Student_grades.objects.filter(
        roll_no=roll_no
    ).order_by('year', 'semester')
```

### Get Verified Grades Only

```python
def get_verified_grades(roll_no):
    """Get only verified grades for a student"""
    return Student_grades.objects.filter(
        roll_no=roll_no,
        verified=True
    ).order_by('year', 'semester')
```

### Get Semester Grades

```python
def get_semester_grades(roll_no, semester, academic_year):
    """Get grades for a specific semester"""
    return Student_grades.objects.filter(
        roll_no=roll_no,
        semester=semester,
        academic_year=academic_year,
        verified=True
    )
```

### Check Backlog Status

```python
def has_backlog(roll_no):
    """Check if student has any backlogs"""
    return Student_grades.objects.filter(
        roll_no=roll_no,
        grade='F',
        verified=True
    ).exists()

def count_backlogs(roll_no):
    """Count number of backlog courses"""
    return Student_grades.objects.filter(
        roll_no=roll_no,
        grade='F',
        verified=True
    ).count()
```

### Get Grade Distribution for a Course

```python
def get_grade_distribution(course_id, semester, year):
    """Get grade distribution for a course"""
    from django.db.models import Count
    
    return Student_grades.objects.filter(
        course_id=course_id,
        semester=semester,
        year=year,
        verified=True
    ).values('grade').annotate(count=Count('grade')).order_by('grade')
```

---

## Relationship Diagram

```
Student_grades
    |
    ├── course_id --> Courses (online_cms local table)
    |                    |
    |                    └── Maps to Course (programme_curriculum)
    |
    └── roll_no --> ExtraInfo.id (globals)
                        |
                        └── Student (academic_information)
```

---

## Sync Points for Other Modules

| Your Module Needs | Use This |
|-------------------|----------|
| Student course grades | Student_grades |
| SGPA calculation | Calculate from Student_grades |
| Backlog check | Filter grade='F' |
| Academic performance | Aggregate grades |
| Grade verification | Check verified=True |

---

## Tables NOT in Production Use

The following tables exist in online_cms but are NOT actively used:

- Modules, CourseDocuments
- AttendanceFiles, Attendance
- Quiz, QuestionBank, Question, QuizResult
- Assignment, StudentAssignment
- Forum, ForumReply
- GradingScheme
- Topics, Courses (local copy)

**Do not integrate with these tables.**

