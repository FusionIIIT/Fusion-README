# Academic Information Module - Integration Guide

**What this module is:** The Academic Information module stores the master student data - every enrolled student's basic academic information including their roll number, batch, programme, category, and hostel details.

**Why it exists:** The Student table is the central identity for all student-related operations in Fusion. Every module that deals with students (Placement, Hostel, Mess, Library, Scholarships) needs to identify and reference students. This module provides that single source of student identity.

**Why integration is needed:** Rather than each module storing duplicate student information, they all link to this module's Student table. When a student's batch changes or they move hostel rooms, it updates in one place and all modules see the updated data.

**Key tables:** Student (linked to ExtraInfo for user details)

---

## Tables (Sync with Production)

### Table: `academic_information_student`

Primary student master table. Links to ExtraInfo via OneToOne relationship.

| Column | Type | Description |
|--------|------|-------------|
| `id_id` | character varying | Primary Key (Roll Number, links to ExtraInfo) |
| `programme` | character varying | Programme code (BTECH, MTECH, PHD) |
| `batch` | integer | Batch year |
| `batch_id_id` | integer | Batch reference from programme_curriculum |
| `category` | character varying | Category (GEN, SC, ST, OBC, EWS) |
| `father_name` | character varying | Father name |
| `mother_name` | character varying | Mother name |
| `hall_no` | integer | Hostel hall number |
| `room_no` | character varying | Room number |
| `specialization` | character varying | M.Tech/PhD specialization |
| `curr_semester_no` | integer | Current semester number |

```sql
SELECT * FROM public.academic_information_student ORDER BY id_id ASC
```

**Usage:** Primary student identification, batch filtering, category-based operations.

**Primary Grades Table:** Use `online_cms_student_grades` for all grade-related operations.

See: [00_Online_CMS_Integration_README.md](00_Online_CMS_Integration_README.md)

---

## How to Use in Your Module

### Import Statements

```python
from applications.academic_information.models import Student
from applications.online_cms.models import Student_grades  # For grades
```

### Common Queries

```python
# Get student by roll number
from applications.globals.models import ExtraInfo
extrainfo = ExtraInfo.objects.get(id='21BCS001')
student = Student.objects.get(id=extrainfo)

# Get student by user
student = Student.objects.get(id__user=user)

# Get all students of a batch
Student.objects.filter(batch_id=batch)

# Get students by category
Student.objects.filter(category='SC')

# Get grades for a student (from online_cms)
from applications.online_cms.models import Student_grades
Student_grades.objects.filter(roll_no='21BCS001', verified=True)
```

### Foreign Key Reference Pattern

```python
# In your module's model
class YourModel(models.Model):
    student = models.ForeignKey(
        'academic_information.Student',
        on_delete=models.CASCADE
    )
```

### Get Student with Related Data

```python
def get_student_complete(roll_no):
    """Get student with all related information"""
    from applications.globals.models import ExtraInfo
    from applications.academic_information.models import Student
    from applications.programme_curriculum.models import Batch
    
    extrainfo = ExtraInfo.objects.select_related('user').get(id=roll_no)
    student = Student.objects.select_related('batch_id').get(id=extrainfo)
    
    return {
        'roll_no': roll_no,
        'name': extrainfo.user.first_name + ' ' + extrainfo.user.last_name,
        'email': extrainfo.user.email,
        'programme': student.programme,
        'batch': student.batch,
        'discipline': student.batch_id.discipline.name,
        'category': student.category,
        'current_semester': student.curr_semester_no
    }
```

---

## Relationship Diagram

```
ExtraInfo (globals)
    |
    └── Student (OneToOne)
            |
            └── batch_id --> Batch (programme_curriculum)
                                |
                                └── discipline --> Discipline

online_cms.Student_grades --> roll_no (matches ExtraInfo.id)
```

---

## Sync Points for Other Modules

| Your Module Needs | Use This Table/Field |
|-------------------|---------------------|
| Student identification | Student.id (via ExtraInfo) |
| Student category | Student.category |
| Batch year | Student.batch or Student.batch_id |
| Programme type | Student.programme |
| Student grades | online_cms.Student_grades |
| Current semester | Student.curr_semester_no |
| Hostel info | Student.hall_no, Student.room_no |

---

## Common Integration Patterns

### Get Students by Discipline

```python
def get_students_by_discipline(discipline_acronym):
    """Get all students of a discipline"""
    return Student.objects.filter(
        batch_id__discipline__acronym=discipline_acronym
    ).select_related('id', 'batch_id')
```

### Check Backlog Status

```python
def has_backlog(student):
    """Check if student has any backlogs"""
    from applications.online_cms.models import Student_grades
    
    failed_grades = Student_grades.objects.filter(
        roll_no=student.id.id,
        grade='F',
        verified=True
    ).count()
    
    return failed_grades > 0
```
