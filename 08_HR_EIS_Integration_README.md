# HR Module (EIS - Employee Information System) - Integration Guide

**What this module is:** The HR/EIS module will manage employee lifecycle - from joining to retirement. It will track service records, qualifications, performance appraisals, training, promotions, leave, and all employee-related administration.

**Why we need to build this:** An institute has faculty, staff, and administrators. Currently HR processes are paper-based - leave applications, appraisals, service records. This module will provide digital employee management with self-service for leave applications and service record viewing.

**Why integration is needed:** Employees exist in globals (ExtraInfo, Faculty, Staff). They belong to departments (Discipline). Faculty teach courses (CourseInstructor). All this data already exists in core modules - HR/EIS will extend it with service-specific information rather than duplicating basic identity data.

**Key dependencies:** ExtraInfo, Faculty, Staff, Discipline, CourseInstructor

---

## Module Overview
The HR/EIS module manages employee records, service history, performance appraisals, training, promotions, and all employee-related administrative functions.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Discipline`
| Column | Type | Description | Usage in HR |
|--------|------|-------------|-------------|
| `id` | AutoField | Primary Key | Faculty discipline reference |
| `name` | CharField(100) | Discipline name | Specialization tracking |

#### Table: `CourseInstructor`
| Column | Type | Description | Usage in HR |
|--------|------|-------------|-------------|
| `instructor_id` | ForeignKey(ExtraInfo) | Instructor | **Teaching workload** |
| `course_id` | ForeignKey(Course) | Course | Course assignment history |
| `year` | IntegerField | Year | Academic workload |

---

### 2. From `academic_information` Module

#### Table: `Curriculum_Instructor`
| Column | Type | Description | Usage in HR |
|--------|------|-------------|-------------|
| `curriculum_id` | ForeignKey(Curriculum) | Curriculum | **Faculty workload tracking** |
| `instructor_id` | ForeignKey(ExtraInfo) | Instructor | Teaching assignment |

---

### 3. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in HR |
|--------|------|-------------|-------------|
| `id` | CharField(20) | Employee ID | **Primary employee identifier** |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | **faculty/staff classification** |
| `sex` | CharField(2) | Gender | Demographics |
| `date_of_birth` | DateField | DOB | **Age calculation, retirement** |
| `user_status` | CharField(50) | Status | **Active/Inactive** |
| `address` | TextField | Address | Employee address |
| `phone_no` | BigIntegerField | Phone | Contact |
| `about_me` | TextField | Bio | Profile |
| `profile_picture` | ImageField | Photo | ID card generation |
| `department` | ForeignKey(DepartmentInfo) | Department | **Department assignment** |

#### Table: `Faculty`
| Column | Type | Description | Usage in HR |
|--------|------|-------------|-------------|
| `id` | OneToOneField(ExtraInfo) | PK | **Faculty reference** |

#### Table: `Staff`
| Column | Type | Description | Usage in HR |
|--------|------|-------------|-------------|
| `id` | OneToOneField(ExtraInfo) | PK | **Staff reference** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in HR |
|--------|------|-------------|-------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | **Employee department** |

#### Table: `Designation`
| Column | Type | Description | Usage in HR |
|--------|------|-------------|-------------|
| `id` | AutoField | Primary Key | Designation reference |
| `name` | CharField(50) | Designation | **Employee designation** |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in HR |
|--------|------|-------------|-------------|
| `id` | AutoField | Primary Key | Record ID |
| `user` | ForeignKey(User) | User | Employee |
| `working` | ForeignKey(ExtraInfo) | ExtraInfo | **Current designation holder** |
| `designation` | ForeignKey(Designation) | Designation | **Current designation** |
| `held_at` | DateTimeField | Held from | **Designation start date** |

---

## Data Models to Create in HR/EIS Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, Faculty, Staff, DepartmentInfo, Designation, HoldsDesignation

# ==================== EMPLOYEE MANAGEMENT ====================

class EmployeeCategory(models.Model):
    """Employee categories"""
    CATEGORY_TYPE = [
        ('TEACHING', 'Teaching'),
        ('NON_TEACHING', 'Non-Teaching'),
    ]
    
    name = models.CharField(max_length=100)  # Professor, Associate Prof, Group-A, etc.
    category_type = models.CharField(max_length=20, choices=CATEGORY_TYPE)
    pay_level = models.CharField(max_length=20, blank=True)  # Level-10, Level-11, etc.
    is_active = models.BooleanField(default=True)

class EmployeeDetails(models.Model):
    """Extended employee information"""
    MARITAL_STATUS = [
        ('SINGLE', 'Single'),
        ('MARRIED', 'Married'),
        ('WIDOWED', 'Widowed'),
        ('DIVORCED', 'Divorced'),
    ]
    
    EMPLOYEE_STATUS = [
        ('ACTIVE', 'Active'),
        ('ON_LEAVE', 'On Leave'),
        ('DEPUTATION', 'On Deputation'),
        ('SUSPENDED', 'Suspended'),
        ('RETIRED', 'Retired'),
        ('RESIGNED', 'Resigned'),
        ('TERMINATED', 'Terminated'),
    ]
    
    # Link to globals
    extra_info = models.OneToOneField(ExtraInfo, on_delete=models.CASCADE, related_name='employee_details')
    
    # Category
    category = models.ForeignKey(EmployeeCategory, on_delete=models.PROTECT)
    
    # Personal
    father_name = models.CharField(max_length=100, blank=True)
    mother_name = models.CharField(max_length=100, blank=True)
    spouse_name = models.CharField(max_length=100, blank=True)
    marital_status = models.CharField(max_length=20, choices=MARITAL_STATUS, blank=True)
    
    # Identifiers
    pan_number = models.CharField(max_length=15, blank=True)
    aadhar_number = models.CharField(max_length=15, blank=True)
    passport_number = models.CharField(max_length=20, blank=True)
    
    # Employment
    date_of_joining = models.DateField(null=True, blank=True)
    date_of_superannuation = models.DateField(null=True, blank=True)
    appointment_type = models.CharField(max_length=50, blank=True)  # Regular, Contract, etc.
    
    # Status
    employee_status = models.CharField(max_length=20, choices=EMPLOYEE_STATUS, default='ACTIVE')
    
    # Bank
    bank_name = models.CharField(max_length=100, blank=True)
    bank_account = models.CharField(max_length=30, blank=True)
    ifsc_code = models.CharField(max_length=15, blank=True)
    
    # Emergency
    emergency_contact_name = models.CharField(max_length=100, blank=True)
    emergency_contact_phone = models.CharField(max_length=15, blank=True)
    emergency_contact_relation = models.CharField(max_length=50, blank=True)

class ServiceHistory(models.Model):
    """Employee service history"""
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='service_history')
    designation = models.ForeignKey(Designation, on_delete=models.PROTECT)
    department = models.ForeignKey(DepartmentInfo, on_delete=models.PROTECT)
    
    from_date = models.DateField()
    to_date = models.DateField(null=True, blank=True)  # Null if current
    
    pay_scale = models.CharField(max_length=50, blank=True)
    basic_pay = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    
    remarks = models.TextField(blank=True)
    
    class Meta:
        ordering = ['-from_date']

# ==================== QUALIFICATION ====================

class QualificationType(models.Model):
    """Types of qualifications"""
    name = models.CharField(max_length=100)  # PhD, MTech, BTech, etc.
    level = models.IntegerField()  # 1=School, 2=UG, 3=PG, 4=Doctoral

class EducationalQualification(models.Model):
    """Employee educational qualifications"""
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='qualifications')
    qualification_type = models.ForeignKey(QualificationType, on_delete=models.PROTECT)
    degree = models.CharField(max_length=100)
    specialization = models.CharField(max_length=200, blank=True)
    institution = models.CharField(max_length=200)
    university = models.CharField(max_length=200, blank=True)
    year_of_passing = models.IntegerField()
    division_grade = models.CharField(max_length=50, blank=True)
    document = models.FileField(upload_to='hr/qualifications/', blank=True)

class ProfessionalQualification(models.Model):
    """Professional certifications"""
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='professional_qualifications')
    title = models.CharField(max_length=200)
    certifying_body = models.CharField(max_length=200)
    date_obtained = models.DateField()
    valid_until = models.DateField(null=True, blank=True)
    document = models.FileField(upload_to='hr/professional/', blank=True)

# ==================== EXPERIENCE ====================

class PreviousExperience(models.Model):
    """Pre-joining experience"""
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='previous_experiences')
    organization = models.CharField(max_length=200)
    designation = models.CharField(max_length=100)
    from_date = models.DateField()
    to_date = models.DateField()
    experience_type = models.CharField(max_length=50)  # Teaching, Industry, Research
    description = models.TextField(blank=True)
    document = models.FileField(upload_to='hr/experience/', blank=True)

# ==================== LEAVE MANAGEMENT ====================

class LeaveType(models.Model):
    """Types of leave"""
    name = models.CharField(max_length=50)  # CL, EL, HPL, etc.
    code = models.CharField(max_length=10, unique=True)
    max_days_per_year = models.IntegerField(null=True, blank=True)
    carry_forward = models.BooleanField(default=False)
    max_carry_forward = models.IntegerField(null=True, blank=True)
    is_active = models.BooleanField(default=True)

class LeaveBalance(models.Model):
    """Employee leave balance"""
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='leave_balances')
    leave_type = models.ForeignKey(LeaveType, on_delete=models.PROTECT)
    year = models.IntegerField()
    opening_balance = models.DecimalField(max_digits=5, decimal_places=1, default=0)
    accrued = models.DecimalField(max_digits=5, decimal_places=1, default=0)
    availed = models.DecimalField(max_digits=5, decimal_places=1, default=0)
    current_balance = models.DecimalField(max_digits=5, decimal_places=1, default=0)
    
    class Meta:
        unique_together = ['employee', 'leave_type', 'year']

class LeaveApplication(models.Model):
    """Leave applications"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('FORWARDED', 'Forwarded'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='leave_applications')
    leave_type = models.ForeignKey(LeaveType, on_delete=models.PROTECT)
    
    from_date = models.DateField()
    to_date = models.DateField()
    num_days = models.DecimalField(max_digits=5, decimal_places=1)
    
    reason = models.TextField()
    address_during_leave = models.TextField(blank=True)
    contact_during_leave = models.CharField(max_length=15, blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    # Workflow
    applied_at = models.DateTimeField(auto_now_add=True)
    forwarded_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='forwarded_leaves')
    forwarded_at = models.DateTimeField(null=True, blank=True)
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='approved_leaves')
    approved_at = models.DateTimeField(null=True, blank=True)
    remarks = models.TextField(blank=True)
    
    # Attachment
    document = models.FileField(upload_to='hr/leave/', blank=True)

# ==================== PERFORMANCE APPRAISAL ====================

class AppraisalPeriod(models.Model):
    """Appraisal periods"""
    name = models.CharField(max_length=100)  # e.g., "2023-24"
    start_date = models.DateField()
    end_date = models.DateField()
    submission_deadline = models.DateField()
    is_active = models.BooleanField(default=False)

class PerformanceAppraisal(models.Model):
    """Employee appraisals"""
    STATUS = [
        ('DRAFT', 'Draft'),
        ('SUBMITTED', 'Submitted'),
        ('REVIEWED', 'Reviewed'),
        ('APPROVED', 'Approved'),
        ('FINALIZED', 'Finalized'),
    ]
    
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='appraisals')
    period = models.ForeignKey(AppraisalPeriod, on_delete=models.PROTECT)
    
    # Self-assessment scores (out of 100)
    teaching_score = models.IntegerField(null=True, blank=True)
    research_score = models.IntegerField(null=True, blank=True)
    admin_score = models.IntegerField(null=True, blank=True)
    extension_score = models.IntegerField(null=True, blank=True)
    
    self_remarks = models.TextField(blank=True)
    
    # Reviewer scores
    reviewer_teaching_score = models.IntegerField(null=True, blank=True)
    reviewer_research_score = models.IntegerField(null=True, blank=True)
    reviewer_admin_score = models.IntegerField(null=True, blank=True)
    reviewer_extension_score = models.IntegerField(null=True, blank=True)
    
    reviewer = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='reviewed_appraisals')
    reviewer_remarks = models.TextField(blank=True)
    
    # Final
    final_score = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)
    final_grade = models.CharField(max_length=10, blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='DRAFT')
    
    submitted_at = models.DateTimeField(null=True, blank=True)
    finalized_at = models.DateTimeField(null=True, blank=True)

# ==================== TRAINING ====================

class TrainingProgram(models.Model):
    """Training programs"""
    title = models.CharField(max_length=200)
    description = models.TextField()
    organizer = models.CharField(max_length=200)
    venue = models.CharField(max_length=200)
    start_date = models.DateField()
    end_date = models.DateField()
    max_participants = models.IntegerField(null=True, blank=True)
    is_mandatory = models.BooleanField(default=False)

class TrainingNomination(models.Model):
    """Employee training nominations"""
    STATUS = [
        ('NOMINATED', 'Nominated'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('ATTENDED', 'Attended'),
        ('COMPLETED', 'Completed'),
    ]
    
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='training_nominations')
    program = models.ForeignKey(TrainingProgram, on_delete=models.CASCADE)
    
    nominated_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='training_nominations_made')
    status = models.CharField(max_length=20, choices=STATUS, default='NOMINATED')
    
    feedback = models.TextField(blank=True)
    certificate = models.FileField(upload_to='hr/training/', blank=True)

# ==================== PROMOTION ====================

class PromotionApplication(models.Model):
    """Promotion applications"""
    STATUS = [
        ('SUBMITTED', 'Submitted'),
        ('UNDER_REVIEW', 'Under Review'),
        ('COMMITTEE_STAGE', 'At Committee'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
    ]
    
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='promotion_applications')
    current_designation = models.ForeignKey(Designation, on_delete=models.PROTECT, related_name='current_for_promotions')
    applied_designation = models.ForeignKey(Designation, on_delete=models.PROTECT, related_name='applied_promotions')
    
    application_date = models.DateField()
    eligibility_date = models.DateField()
    
    # Documents
    api_score = models.IntegerField(null=True, blank=True)  # For faculty
    documents = models.FileField(upload_to='hr/promotions/', blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='SUBMITTED')
    remarks = models.TextField(blank=True)
    
    # Result
    approved_date = models.DateField(null=True, blank=True)
    effective_date = models.DateField(null=True, blank=True)

# ==================== ATTENDANCE ====================

class EmployeeAttendance(models.Model):
    """Daily attendance"""
    ATTENDANCE_STATUS = [
        ('PRESENT', 'Present'),
        ('ABSENT', 'Absent'),
        ('HALF_DAY', 'Half Day'),
        ('ON_LEAVE', 'On Leave'),
        ('ON_TOUR', 'On Tour'),
        ('WORK_FROM_HOME', 'WFH'),
    ]
    
    employee = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='attendance_records')
    date = models.DateField()
    status = models.CharField(max_length=20, choices=ATTENDANCE_STATUS)
    
    in_time = models.TimeField(null=True, blank=True)
    out_time = models.TimeField(null=True, blank=True)
    
    remarks = models.TextField(blank=True)
    
    class Meta:
        unique_together = ['employee', 'date']

# ==================== FACULTY WORKLOAD ====================

class FacultyWorkload(models.Model):
    """Faculty teaching workload per semester"""
    faculty = models.ForeignKey(Faculty, on_delete=models.CASCADE, related_name='workloads')
    semester = models.CharField(max_length=20)  # e.g., "2024-Spring"
    year = models.IntegerField()
    
    # Hours per week
    lecture_hours = models.IntegerField(default=0)
    tutorial_hours = models.IntegerField(default=0)
    lab_hours = models.IntegerField(default=0)
    total_hours = models.IntegerField(default=0)
    
    # Students
    total_students = models.IntegerField(default=0)
    phd_scholars = models.IntegerField(default=0)
    
    class Meta:
        unique_together = ['faculty', 'semester', 'year']
```

---

## Integration Functions

### Calculate Faculty Workload
```python
def calculate_faculty_workload(faculty_extra_info, semester, year):
    """Calculate teaching workload from course assignments"""
    from applications.programme_curriculum.models import CourseInstructor
    from applications.academic_information.models import Curriculum_Instructor
    
    # Get courses assigned
    course_assignments = CourseInstructor.objects.filter(
        instructor_id=faculty_extra_info,
        year=year
    ).select_related('course_id')
    
    total_hours = 0
    total_students = 0
    
    for assignment in course_assignments:
        course = assignment.course_id
        total_hours += course.lecture_hours + course.tutorial_hours + course.pratical_hours
        # Count students registered
    
    return {
        'total_hours': total_hours,
        'total_students': total_students
    }
```

### Get Employee's Current Designation
```python
def get_current_designation(employee_extra_info):
    """Get current designation of employee"""
    from applications.globals.models import HoldsDesignation
    
    held = HoldsDesignation.objects.filter(
        working=employee_extra_info
    ).order_by('-held_at').first()
    
    if held:
        return held.designation
    return None
```

### Calculate Retirement Date
```python
def calculate_retirement_date(employee_extra_info, retirement_age=60):
    """Calculate superannuation date"""
    from datetime import date
    from dateutil.relativedelta import relativedelta
    
    dob = employee_extra_info.date_of_birth
    if dob:
        retirement_date = dob + relativedelta(years=retirement_age)
        return retirement_date
    return None
```

### Get All Employees by Department
```python
def get_department_employees(department_id, employee_type=None):
    """Get all employees in a department"""
    from applications.globals.models import ExtraInfo
    
    filters = {'department_id': department_id}
    if employee_type:
        filters['user_type'] = employee_type
    
    return ExtraInfo.objects.filter(**filters).select_related('user')
```

### Check Leave Eligibility
```python
def check_leave_eligibility(employee_extra_info, leave_type, num_days):
    """Check if employee can apply for leave"""
    from datetime import date
    
    # Get current balance
    current_year = date.today().year
    balance = LeaveBalance.objects.filter(
        employee=employee_extra_info,
        leave_type=leave_type,
        year=current_year
    ).first()
    
    if balance and balance.current_balance >= num_days:
        return True, balance.current_balance
    return False, balance.current_balance if balance else 0
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/hr/api/employees/` | GET | Employee list |
| `/hr/api/employees/<id>/` | GET, PUT | Employee details |
| `/hr/api/employees/<id>/service-history/` | GET | Service history |
| `/hr/api/leave-applications/` | GET, POST | Leave applications |
| `/hr/api/leave-balance/<id>/` | GET | Leave balance |
| `/hr/api/appraisals/` | GET, POST | Performance appraisals |
| `/hr/api/training/` | GET | Training programs |
| `/hr/api/promotions/` | GET, POST | Promotion applications |

---

## Important Sync Points

1. **Employee Base**: Always use `ExtraInfo` as base; extend with `EmployeeDetails`.
2. **Department**: Get from `ExtraInfo.department` (ForeignKey to DepartmentInfo).
3. **Designation**: Current designation from `HoldsDesignation`, history in `ServiceHistory`.
4. **Faculty/Staff**: Use `Faculty`/`Staff` models to distinguish teaching vs non-teaching.
5. **Workload**: Calculate from `CourseInstructor` and `Curriculum_Instructor`.
6. **Leave Module**: Integrate with existing `leave` module if present.

---

