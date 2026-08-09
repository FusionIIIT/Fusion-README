# LMS (Learning Management System) - Integration Guide

**What this module is:** The LMS (Learning Management System) will handle online course delivery - managing lecture materials, uploading resources, tracking attendance, conducting quizzes, handling assignments, and facilitating discussions.

**Why we need to build this:** Currently there's no integrated digital learning platform. Faculty share materials via email or external drives. Assignments are submitted physically. Quizzes happen on paper. This module will provide a unified online classroom - content sharing, assignment submission, quiz conducting, and progress tracking.

**Why integration is needed:** LMS must know which student is enrolled in which course (from course_registration), who teaches which course (from CourseInstructor), and student identity (from Student table). Without this production data, LMS cannot show the right content to the right user.

**Key dependencies:** Student, course_registration, Courses, CourseInstructor, ExtraInfo

---

## Module Overview

The LMS module will provide a complete online learning platform for the institute. It will enable faculty to share course materials, conduct online quizzes, manage assignments, and track student engagement.

---

## Core Dependencies - Tables to Sync With

### From `programme_curriculum` Module

| Table                | Key Columns              | Usage                            |
| -------------------- | ------------------------ | -------------------------------- |
| `Course`           | id, code, name           | Course reference for LMS content |
| `CourseInstructor` | course_id, instructor_id | Identify who teaches what        |

### From `academic_information` Module

| Table       | Key Columns  | Usage                  |
| ----------- | ------------ | ---------------------- |
| `Student` | id, batch_id | Student identification |

### From `academic_procedures` Module

| Table                   | Key Columns           | Usage                           |
| ----------------------- | --------------------- | ------------------------------- |
| `course_registration` | student_id, course_id | Who is enrolled in which course |

### From `globals` Module

| Table         | Key Columns         | Usage                  |
| ------------- | ------------------- | ---------------------- |
| `ExtraInfo` | id, user, user_type | User identification    |
| `Faculty`   | id, extra_info_id   | Faculty identification |

---

## Proposed Models (To Be Built)

### Course Content

```python
class LMSModule(models.Model):
    """Content modules within a course"""
    course = ForeignKey(Courses, on_delete=CASCADE)
    title = CharField(max_length=255)
    description = TextField(null=True)
    order = IntegerField()
    is_visible = BooleanField(default=True)

class LMSContent(models.Model):
    """Individual content items (files, videos, etc.)"""
    module = ForeignKey(LMSModule, on_delete=CASCADE)
    title = CharField(max_length=255)
    content_type = CharField(max_length=50)  # pdf, video, link
    file = FileField(null=True)
    url = URLField(null=True)
    uploaded_by = ForeignKey(ExtraInfo, on_delete=CASCADE)
    uploaded_at = DateTimeField(auto_now_add=True)
```

### Assignments

```python
class LMSAssignment(models.Model):
    """Assignment definition"""
    course = ForeignKey(Courses, on_delete=CASCADE)
    title = CharField(max_length=255)
    description = TextField()
    due_date = DateTimeField()
    max_marks = IntegerField()
    created_by = ForeignKey(ExtraInfo, on_delete=CASCADE)

class LMSSubmission(models.Model):
    """Student assignment submissions"""
    assignment = ForeignKey(LMSAssignment, on_delete=CASCADE)
    student = ForeignKey(Student, on_delete=CASCADE)
    submitted_at = DateTimeField(auto_now_add=True)
    file = FileField()
    marks = IntegerField(null=True)
    feedback = TextField(null=True)
```

### Quizzes

```python
class LMSQuiz(models.Model):
    """Quiz definition"""
    course = ForeignKey(Courses, on_delete=CASCADE)
    title = CharField(max_length=255)
    duration_minutes = IntegerField()
    start_time = DateTimeField()
    end_time = DateTimeField()
    max_marks = IntegerField()

class LMSQuestion(models.Model):
    """Quiz questions"""
    quiz = ForeignKey(LMSQuiz, on_delete=CASCADE)
    question_text = TextField()
    question_type = CharField(max_length=20)  # mcq, short, long
    marks = IntegerField()
    options = JSONField(null=True)  # For MCQ
    correct_answer = TextField(null=True)
```

---

## Integration Functions (To Be Implemented)

### Get Enrolled Students for a Course

```python
from applications.academic_procedures.models import course_registration

def get_enrolled_students(course_id, semester_id):
    """Get all students enrolled in a course"""
    return course_registration.objects.filter(
        course_id=course_id,
        semester_id=semester_id
    ).select_related('student_id')
```

### Get Courses for a Faculty

```python
from applications.programme_curriculum.models import CourseInstructor

def get_faculty_courses(faculty_id, semester_id):
    """Get all courses taught by a faculty"""
    return CourseInstructor.objects.filter(
        instructor_id=faculty_id,
        semester=semester_id
    ).select_related('course_id')
```

### Get Student's Courses

```python
def get_student_courses(student_id, semester_id):
    """Get all courses a student is enrolled in"""
    return course_registration.objects.filter(
        student_id=student_id,
        semester_id=semester_id
    ).select_related('course_id')
```

---

## Integration with Other Modules

| Module               | Integration Point                               |
| -------------------- | ----------------------------------------------- |
| Academic Procedures  | Read course_registration to know enrollments    |
| Programme Curriculum | Read Course and CourseInstructor data           |
| Globals              | Read ExtraInfo, Faculty for user identification |
| Notifications        | Send notifications for new content, deadlines   |
| Dashboards           | Show pending assignments, upcoming quizzes      |
