# Department Module - Integration Guide

**What this module is:** The Department module will handle department-level operations - managing faculty within a department, department announcements, research coordination, curriculum oversight, and department-specific student interactions.

**Why we need to build this:** Each academic department (CSE, ECE, ME, etc.) has administrative needs but currently HODs lack a unified view of their domain. This module will provide department-scoped views - their students, faculty workload, course offerings, and research activities in one place.

**Why integration is needed:** Department data comes from core modules. Students belong to batches which belong to disciplines (departments). Faculty teach courses. Course instructors are assigned per curriculum. All this data exists in programme_curriculum and academic_information - the Department module will query these rather than duplicating.

**Key dependencies:** Discipline, Batch, Student, Course, CourseInstructor, Faculty

---

## Module Overview
The Department module manages department-specific operations including faculty management, department announcements, research activities, department meetings, curriculum oversight, and student-faculty interactions at the department level.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Programme`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | AutoField | Primary Key | Programme reference |
| `category` | CharField(3) | UG/PG/PHD | Programme categorization |
| `name` | CharField(70) | Programme name | Department programmes |
| `programme_begin_year` | PositiveIntegerField | Start year | Historical tracking |

#### Table: `Discipline`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | AutoField | Primary Key | **Primary department identifier** |
| `name` | CharField(100) | Discipline name | Department name mapping |
| `acronym` | CharField(10) | Short code (CSE, ECE, ME) | Display and filtering |
| `programmes` | ManyToManyField(Programme) | Linked programmes | Programmes under department |

#### Table: `Curriculum`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | AutoField | Primary Key | Curriculum reference |
| `programme` | ForeignKey(Programme) | Programme | Programme curriculum |
| `name` | CharField(100) | Curriculum name | Display purposes |
| `version` | DecimalField | Version number | Version tracking |
| `working_curriculum` | BooleanField | Is active | Active curriculum filter |
| `no_of_semester` | PositiveIntegerField | Total semesters | Semester planning |

#### Table: `Course`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | AutoField | Primary Key | Course reference |
| `code` | CharField(10) | Course code | Department courses |
| `name` | CharField(100) | Course name | Display purposes |
| `credit` | PositiveIntegerField | Credits | Credit management |
| `disciplines` | ManyToManyField(Discipline) | Linked disciplines | **Department course mapping** |
| `working_course` | BooleanField | Is active | Active course filter |

#### Table: `CourseInstructor`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | AutoField | Primary Key | Reference |
| `course_id` | ForeignKey(Course) | Course | Course assignment |
| `instructor_id` | ForeignKey(Faculty) | Faculty | **Faculty workload** |
| `year` | IntegerField | Academic year | Year filtering |
| `semester_type` | CharField(20) | Semester type | Semester filtering |

#### Table: `Batch`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | AutoField | Primary Key | Batch reference |
| `name` | CharField(50) | Batch name | Batch identification |
| `discipline` | ForeignKey(Discipline) | **Discipline** | **Department's batches** |
| `year` | PositiveIntegerField | Batch year | Year filtering |
| `running_batch` | BooleanField | Is active | Active batch filter |
| `total_seats` | PositiveIntegerField | Seat count | **Intake statistics** |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | Student reference |
| `programme` | CharField(10) | Programme | Programme filtering |
| `batch` | IntegerField | Batch year | Batch statistics |
| `batch_id` | ForeignKey(Batch) | Batch reference | **Department's students** |
| `specialization` | CharField(40) | Specialization | Specialization stats |
| `curr_semester_no` | IntegerField | Current semester | Semester distribution |

---

### 3. From `academic_procedures` Module

#### Table: `course_registration`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `student_id` | ForeignKey(Student) | Student | Enrollment data |
| `course_id` | ForeignKey(Course) | Course | Course enrollment |
| `semester_id` | ForeignKey(Semester) | Semester | Semester stats |

#### Table: `ThesisTopicProcess`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `student_id` | ForeignKey(Student) | Student | Thesis student |
| `supervisor_id` | ForeignKey(Faculty) | Supervisor | **Faculty research load** |
| `thesis_topic` | CharField | Topic | Research tracking |
| `approval_by_hod` | BooleanField | HOD approval | **Department approval** |

---

### 4. From `globals` Module

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | AutoField | Primary Key | **Department identifier** |
| `name` | CharField(100) | Department name | **Primary department reference** |

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | CharField(20) | User ID | User identification |
| `user` | OneToOneField(User) | Django User | User reference |
| `user_type` | CharField(20) | User type | Role filtering |
| `department` | ForeignKey(DepartmentInfo) | **Department** | **User-Department mapping** |

#### Table: `Faculty`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Department faculty** |

#### Table: `Designation`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `id` | AutoField | Primary Key | Designation reference |
| `name` | CharField(50) | Designation name | Role identification |
| `full_name` | CharField(100) | Full name | Display purposes |
| `type` | CharField(30) | academic/administrative | Designation type |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in Department |
|--------|------|-------------|---------------------|
| `user` | ForeignKey(User) | User | Designation holder |
| `working` | ForeignKey(User) | Active holder | Current designation |
| `designation` | ForeignKey(Designation) | Designation | **HOD identification** |
| `held_at` | DateTimeField | Timestamp | Assignment date |

---

## Key Relationships

### Department to Discipline Mapping
```python
# DepartmentInfo maps to Discipline
# Example: DepartmentInfo.name = "Computer Science and Engineering" 
#          Discipline.name = "Computer Science and Engineering"
#          Discipline.acronym = "CSE"

def get_discipline_for_department(department_name):
    """Get discipline matching department"""
    from applications.programme_curriculum.models import Discipline
    return Discipline.objects.filter(
        name__icontains=department_name.replace('department:', '').strip()
    ).first()
```

### HOD Identification
```python
def get_department_hod(department):
    """Get HOD for a department"""
    from applications.globals.models import HoldsDesignation, Designation
    
    # HOD designation pattern: "HOD (CSE)", "HOD (ECE)", etc.
    hod_designation = HoldsDesignation.objects.filter(
        designation__name__icontains='hod'
    ).select_related('working', 'designation')
    
    for hd in hod_designation:
        if department.name.lower() in hd.designation.name.lower():
            return hd.working
    return None
```

---

## Data Models to Create in Department Module

```python
from django.db import models
from applications.globals.models import DepartmentInfo, ExtraInfo, Faculty
from applications.programme_curriculum.models import Discipline, Course, Batch

class DepartmentProfile(models.Model):
    """Extended department information"""
    department = models.OneToOneField(DepartmentInfo, on_delete=models.CASCADE)
    discipline = models.ForeignKey(Discipline, on_delete=models.SET_NULL, null=True)
    
    # Department details
    vision = models.TextField(blank=True)
    mission = models.TextField(blank=True)
    about = models.TextField(blank=True)
    establishment_year = models.IntegerField(null=True)
    
    # Contact
    office_location = models.CharField(max_length=100, blank=True)
    phone = models.CharField(max_length=20, blank=True)
    email = models.EmailField(blank=True)
    website = models.URLField(blank=True)
    
    # Statistics (updated periodically)
    total_faculty = models.IntegerField(default=0)
    total_students = models.IntegerField(default=0)
    total_phd_scholars = models.IntegerField(default=0)
    
    updated_at = models.DateTimeField(auto_now=True)

class DepartmentAnnouncement(models.Model):
    """Department-level announcements"""
    ANNOUNCEMENT_TYPE = [
        ('GENERAL', 'General'),
        ('ACADEMIC', 'Academic'),
        ('PLACEMENT', 'Placement'),
        ('EVENT', 'Event'),
        ('URGENT', 'Urgent'),
    ]
    
    department = models.ForeignKey(DepartmentInfo, on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    content = models.TextField()
    announcement_type = models.CharField(max_length=20, choices=ANNOUNCEMENT_TYPE)
    created_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    valid_until = models.DateTimeField(null=True, blank=True)
    is_active = models.BooleanField(default=True)
    attachment = models.FileField(upload_to='department/announcements/', null=True, blank=True)
    
    # Target audience
    for_students = models.BooleanField(default=True)
    for_faculty = models.BooleanField(default=True)
    for_staff = models.BooleanField(default=False)
    for_batches = models.ManyToManyField(Batch, blank=True)

class DepartmentMeeting(models.Model):
    """Department meetings"""
    MEETING_TYPE = [
        ('FACULTY', 'Faculty Meeting'),
        ('DAC', 'Department Academic Committee'),
        ('DPGC', 'Department PG Committee'),
        ('DUGC', 'Department UG Committee'),
        ('SPECIAL', 'Special Meeting'),
    ]
    
    department = models.ForeignKey(DepartmentInfo, on_delete=models.CASCADE)
    meeting_type = models.CharField(max_length=20, choices=MEETING_TYPE)
    title = models.CharField(max_length=200)
    date = models.DateField()
    time = models.TimeField()
    venue = models.CharField(max_length=100)
    agenda = models.TextField()
    
    # Participants
    convener = models.ForeignKey(Faculty, on_delete=models.SET_NULL, null=True, related_name='convened_meetings')
    attendees = models.ManyToManyField(Faculty, related_name='department_meetings')
    
    # Minutes
    minutes = models.TextField(blank=True)
    minutes_file = models.FileField(upload_to='department/meetings/', null=True, blank=True)
    minutes_approved = models.BooleanField(default=False)
    
    created_at = models.DateTimeField(auto_now_add=True)

class FacultyDepartmentAssignment(models.Model):
    """Track faculty assignment to departments"""
    faculty = models.ForeignKey(Faculty, on_delete=models.CASCADE)
    department = models.ForeignKey(DepartmentInfo, on_delete=models.CASCADE)
    is_primary = models.BooleanField(default=True)  # Primary department
    joined_date = models.DateField()
    left_date = models.DateField(null=True, blank=True)
    designation_in_dept = models.CharField(max_length=100, blank=True)  # HOD, Faculty, etc.
    
    class Meta:
        unique_together = ['faculty', 'department', 'joined_date']

class DepartmentCommittee(models.Model):
    """Department committees"""
    COMMITTEE_TYPE = [
        ('DAC', 'Department Academic Committee'),
        ('DPGC', 'Department PG Committee'),
        ('DUGC', 'Department UG Committee'),
        ('RESEARCH', 'Research Committee'),
        ('PLACEMENT', 'Placement Committee'),
        ('OTHER', 'Other'),
    ]
    
    department = models.ForeignKey(DepartmentInfo, on_delete=models.CASCADE)
    name = models.CharField(max_length=100)
    committee_type = models.CharField(max_length=20, choices=COMMITTEE_TYPE)
    chairman = models.ForeignKey(Faculty, on_delete=models.SET_NULL, null=True, related_name='chaired_committees')
    members = models.ManyToManyField(Faculty, related_name='committee_memberships')
    formation_date = models.DateField()
    tenure_end = models.DateField(null=True, blank=True)
    responsibilities = models.TextField(blank=True)
    is_active = models.BooleanField(default=True)

class DepartmentResearchArea(models.Model):
    """Research areas of the department"""
    department = models.ForeignKey(DepartmentInfo, on_delete=models.CASCADE)
    area_name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    lead_faculty = models.ManyToManyField(Faculty, related_name='research_areas')
    is_active = models.BooleanField(default=True)
```

---

## Integration Queries

### Get Department Statistics
```python
def get_department_statistics(department_id):
    """Get comprehensive department statistics"""
    from applications.globals.models import DepartmentInfo, ExtraInfo, Faculty
    from applications.academic_information.models import Student
    from applications.programme_curriculum.models import Discipline, Batch, Course
    
    department = DepartmentInfo.objects.get(id=department_id)
    
    # Get faculty count
    faculty_count = ExtraInfo.objects.filter(
        department=department,
        user_type='faculty'
    ).count()
    
    # Get discipline
    discipline = Discipline.objects.filter(
        name__icontains=department.name.replace('department:', '').strip()
    ).first()
    
    # Get batches and students
    batches = []
    student_count = 0
    if discipline:
        batches = Batch.objects.filter(discipline=discipline, running_batch=True)
        for batch in batches:
            student_count += Student.objects.filter(batch_id=batch).count()
    
    # Get courses
    course_count = 0
    if discipline:
        course_count = Course.objects.filter(
            disciplines=discipline,
            working_course=True
        ).count()
    
    return {
        'department': department,
        'discipline': discipline,
        'faculty_count': faculty_count,
        'student_count': student_count,
        'active_batches': batches.count() if batches else 0,
        'courses_offered': course_count,
    }
```

### Get Department Faculty
```python
def get_department_faculty(department_id):
    """Get all faculty in a department"""
    from applications.globals.models import DepartmentInfo, ExtraInfo, Faculty
    
    department = DepartmentInfo.objects.get(id=department_id)
    
    faculty_extrainfos = ExtraInfo.objects.filter(
        department=department,
        user_type='faculty'
    ).select_related('user')
    
    faculty_list = []
    for extra in faculty_extrainfos:
        try:
            faculty = Faculty.objects.get(id=extra)
            faculty_list.append({
                'faculty': faculty,
                'extrainfo': extra,
                'name': f"{extra.user.first_name} {extra.user.last_name}",
                'email': extra.user.email,
            })
        except Faculty.DoesNotExist:
            pass
    
    return faculty_list
```

### Get Department Students
```python
def get_department_students(department_id, batch_year=None):
    """Get all students in a department"""
    from applications.globals.models import DepartmentInfo
    from applications.academic_information.models import Student
    from applications.programme_curriculum.models import Discipline, Batch
    
    department = DepartmentInfo.objects.get(id=department_id)
    
    # Find discipline
    discipline = Discipline.objects.filter(
        name__icontains=department.name.replace('department:', '').strip()
    ).first()
    
    if not discipline:
        return []
    
    # Get batches
    batches = Batch.objects.filter(discipline=discipline)
    if batch_year:
        batches = batches.filter(year=batch_year)
    
    # Get students
    students = Student.objects.filter(
        batch_id__in=batches
    ).select_related('id', 'id__user', 'batch_id')
    
    return students
```

### Get Faculty Workload
```python
def get_faculty_workload(faculty_id, year=None, semester_type=None):
    """Get teaching workload for a faculty"""
    from applications.programme_curriculum.models import CourseInstructor, Course
    from applications.academic_procedures.models import ThesisTopicProcess
    from applications.globals.models import Faculty
    
    faculty = Faculty.objects.get(id=faculty_id)
    
    # Get courses taught
    course_filter = {'instructor_id': faculty}
    if year:
        course_filter['year'] = year
    if semester_type:
        course_filter['semester_type'] = semester_type
    
    courses = CourseInstructor.objects.filter(**course_filter).select_related('course_id')
    
    # Calculate teaching hours
    total_lecture_hours = 0
    total_tutorial_hours = 0
    total_practical_hours = 0
    
    for ci in courses:
        course = ci.course_id
        total_lecture_hours += course.lecture_hours or 0
        total_tutorial_hours += course.tutorial_hours or 0
        total_practical_hours += course.pratical_hours or 0
    
    # Get thesis supervision count
    thesis_count = ThesisTopicProcess.objects.filter(
        supervisor_id=faculty,
        approval_supervisor=True
    ).count()
    
    return {
        'faculty': faculty,
        'courses_count': courses.count(),
        'courses': list(courses),
        'total_lecture_hours': total_lecture_hours,
        'total_tutorial_hours': total_tutorial_hours,
        'total_practical_hours': total_practical_hours,
        'thesis_supervision_count': thesis_count,
    }
```

---

## API Endpoints Required

### From Core Modules
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/programme_curriculum/api/disciplines/` | GET | Get disciplines |
| `/programme_curriculum/api/courses/` | GET | Get courses |
| `/programme_curriculum/api/batches/` | GET | Get batches |
| `/programme_curriculum/api/course-instructors/` | GET | Faculty assignments |

### New Endpoints to Create
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/department/api/profile/<id>/` | GET, PUT | Department profile |
| `/department/api/faculty/<dept_id>/` | GET | Department faculty |
| `/department/api/students/<dept_id>/` | GET | Department students |
| `/department/api/announcements/` | GET, POST | Announcements |
| `/department/api/meetings/` | GET, POST | Meetings |
| `/department/api/committees/` | GET, POST | Committees |
| `/department/api/statistics/<dept_id>/` | GET | Department stats |
| `/department/api/faculty-workload/<faculty_id>/` | GET | Faculty workload |

---

## Views Integration Pattern

```python
from django.contrib.auth.decorators import login_required
from applications.globals.models import ExtraInfo, HoldsDesignation

@login_required
def department_dashboard(request):
    """Department dashboard view"""
    user = request.user
    extrainfo = ExtraInfo.objects.get(user=user)
    
    # Check if user is HOD
    is_hod = HoldsDesignation.objects.filter(
        working=user,
        designation__name__icontains='hod'
    ).exists()
    
    # Get user's department
    department = extrainfo.department
    
    if not department:
        return HttpResponseRedirect('/dashboard/')
    
    # Get department statistics
    stats = get_department_statistics(department.id)
    
    # Get announcements
    announcements = DepartmentAnnouncement.objects.filter(
        department=department,
        is_active=True
    ).order_by('-created_at')[:10]
    
    # Get upcoming meetings (for faculty)
    upcoming_meetings = []
    if extrainfo.user_type == 'faculty':
        upcoming_meetings = DepartmentMeeting.objects.filter(
            department=department,
            date__gte=timezone.now().date()
        ).order_by('date', 'time')[:5]
    
    return render(request, 'department/dashboard.html', {
        'department': department,
        'stats': stats,
        'announcements': announcements,
        'upcoming_meetings': upcoming_meetings,
        'is_hod': is_hod,
    })
```

---

## Important Sync Points

1. **Department-Discipline Mapping**: Ensure `DepartmentInfo.name` matches `Discipline.name` for proper linking.

2. **HOD Detection**: Use `HoldsDesignation` with designation name containing 'hod' to identify HOD.

3. **Faculty Assignment**: Track via `ExtraInfo.department` field.

4. **Course Ownership**: Use `Course.disciplines` ManyToMany field.

5. **Batch Ownership**: Use `Batch.discipline` foreign key.

6. **Student Assignment**: Students belong to department via `Student.batch_id.discipline`.

---

