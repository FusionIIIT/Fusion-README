# Award and Scholarship Module - Integration Guide

**What this module is:** The Award and Scholarship module will manage all financial aid and recognition programs - merit scholarships, need-based aid, government scholarships, sports awards, and academic excellence recognitions.

**Why we need to build this:** Students receive various scholarships but currently eligibility is verified manually using spreadsheets. This module will automate the entire scholarship lifecycle - applications, eligibility checks against academic records, approvals, and disbursement tracking.

**Why integration is needed:** Scholarship eligibility depends on data from core modules. Need to verify student's batch and programme (from programme_curriculum), check if they have backlogs (from Student_grades), verify fee payment status (from FeePayments), and check dues clearance (from Dues). Without integration, eligibility checks would be impossible.

**Key dependencies:** Student, Batch, Programme, Student_grades, FeePayments, Dues

---

## Module Overview
The Award and Scholarship module manages student scholarships, awards, merit-based recognitions, financial aid, and fellowship programs within the institute.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Programme`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `id` | AutoField | Primary Key | Programme identification |
| `category` | CharField(3) | UG/PG/PHD | Scholarship eligibility criteria |
| `name` | CharField(70) | Programme name | Display and filtering |
| `programme_begin_year` | PositiveIntegerField | Start year | Historical tracking |

#### Table: `Batch`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `id` | AutoField | Primary Key | Batch reference |
| `name` | CharField(50) | B.Tech/M.Tech etc. | Scholarship category |
| `discipline` | ForeignKey(Discipline) | Discipline | Department-wise awards |
| `year` | PositiveIntegerField | Batch year | Year-based scholarships |
| `curriculum` | ForeignKey(Curriculum) | Curriculum | Academic tracking |
| `running_batch` | BooleanField | Is active | Active student filtering |

#### Table: `Discipline`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `id` | AutoField | Primary Key | Discipline identification |
| `name` | CharField(100) | Discipline name | Department-specific awards |
| `acronym` | CharField(10) | Short code (CSE, ECE) | Display purposes |
| `programmes` | ManyToManyField(Programme) | Linked programmes | Programme filtering |

#### Table: `Course`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `id` | AutoField | Primary Key | Course reference |
| `code` | CharField(10) | Course code | Course-specific awards |
| `name` | CharField(100) | Course name | Display purposes |
| `credit` | PositiveIntegerField | Credits | Credit-based eligibility |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key (Roll No) | **Primary recipient identification** |
| `programme` | CharField(10) | B.Tech/M.Tech/PhD | Scholarship eligibility |
| `batch` | IntegerField | Batch year | Year-based filtering |
| `batch_id` | ForeignKey(Batch) | Batch reference | Batch linking |
| `category` | CharField(10) | GEN/SC/ST/OBC | **Category-based scholarships** |
| `specialization` | CharField(40) | M.Tech specialization | Specialization awards |
| `curr_semester_no` | IntegerField | Current semester | Semester eligibility |

#### Table: `Student_grades` (from online_cms)
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `roll_no` | TextField | Student roll number | Grade tracking |
| `course_id` | ForeignKey | Course reference | Course performance |
| `grade` | TextField | Grade (A/A-/B+...) | **Academic excellence awards** |
| `verified` | BooleanField | Verification status | Verified grades only |

---

### 3. From `academic_procedures` Module

#### Table: `course_registration`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `student_id` | ForeignKey(Student) | Student reference | Enrollment verification |
| `semester_id` | ForeignKey(Semester) | Semester | Active semester |
| `course_id` | ForeignKey(Course) | Course | Course completion |
| `registration_type` | CharField(20) | Regular/Backlog | Backlog count for eligibility |
| `session` | CharField(9) | Academic session | Session tracking |

#### Table: `FeePayments`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `student_id` | ForeignKey(Student) | Student reference | Fee status |
| `semester_id` | ForeignKey(Semester) | Semester | Semester tracking |
| `fee_paid` | IntegerField | Amount paid | **Financial aid calculation** |
| `actual_fee` | IntegerField | Actual fee | Fee waiver calculation |
| `transaction_id` | CharField(40) | Transaction ID | Payment verification |

#### Table: `Dues`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `student_id` | ForeignKey(Student) | Student reference | Due tracking |
| `mess_due` | IntegerField | Mess dues | Due clearance check |
| `hostel_due` | IntegerField | Hostel dues | Due clearance check |
| `library_due` | IntegerField | Library dues | Due clearance check |
| `academic_due` | IntegerField | Academic dues | **Scholarship disbursement eligibility** |

---

### 4. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `id` | CharField(20) | User ID/Roll No | **Primary identification** |
| `user` | OneToOneField(User) | Django User | User reference |
| `date_of_birth` | DateField | DOB | Age-based scholarships |
| `address` | TextField | Address | Communication |
| `phone_no` | BigIntegerField | Phone | Communication |
| `department` | ForeignKey(DepartmentInfo) | Department | Department filtering |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in Awards/Scholarship |
|--------|------|-------------|---------------------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | Department-wise awards |

---

## Data Required for Integration

### Student Eligibility Criteria Queries

```python
# Get students by category for category-based scholarships
def get_category_students(category, batch=None):
    queryset = Student.objects.filter(category=category)
    if batch:
        queryset = queryset.filter(batch=batch)
    return queryset

# Get students with no backlogs
def get_students_without_backlogs(batch):
    from django.db.models import Count, Q
    return Student.objects.filter(batch=batch).annotate(
        backlog_count=Count(
            'course_registration',
            filter=Q(course_registration__registration_type='Backlog')
        )
    ).filter(backlog_count=0)
```

### Academic Performance Queries

```python
# Get students with specific grade in a course
def get_course_toppers(course_id, grade='A'):
    from applications.online_cms.models import Student_grades
    return Student_grades.objects.filter(
        course_id=course_id,
        grade=grade,
        verified=True
    )
```

---

## Models to Create in Award and Scholarship Module

```python
from django.db import models
from applications.academic_information.models import Student
from applications.programme_curriculum.models import Programme, Batch, Discipline
from applications.globals.models import ExtraInfo

class ScholarshipType(models.Model):
    """Define types of scholarships available"""
    CATEGORY_CHOICES = [
        ('MERIT', 'Merit-based'),
        ('NEED', 'Need-based'),
        ('CATEGORY', 'Category-based'),
        ('SPORTS', 'Sports'),
        ('CULTURAL', 'Cultural'),
        ('RESEARCH', 'Research'),
        ('EXTERNAL', 'External/Government'),
    ]
    
    name = models.CharField(max_length=200)
    category = models.CharField(max_length=20, choices=CATEGORY_CHOICES)
    description = models.TextField()
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    frequency = models.CharField(max_length=20)  # Monthly/Semester/Annual
    eligibility_criteria = models.TextField()
    max_backlogs = models.IntegerField(default=0)
    applicable_programmes = models.ManyToManyField(Programme, blank=True)
    applicable_batches = models.ManyToManyField(Batch, blank=True)
    applicable_categories = models.CharField(max_length=50, blank=True)  # GEN,SC,ST,OBC
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

class ScholarshipApplication(models.Model):
    """Student scholarship applications"""
    STATUS_CHOICES = [
        ('PENDING', 'Pending'),
        ('UNDER_REVIEW', 'Under Review'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('DISBURSED', 'Disbursed'),
    ]
    
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    scholarship_type = models.ForeignKey(ScholarshipType, on_delete=models.CASCADE)
    academic_year = models.CharField(max_length=9)  # e.g., 2024-25
    semester = models.IntegerField()
    
    # Auto-fetched from Student model
    category_at_application = models.CharField(max_length=10)
    
    # Application details
    application_date = models.DateTimeField(auto_now_add=True)
    supporting_documents = models.FileField(upload_to='scholarships/documents/', null=True)
    remarks = models.TextField(blank=True)
    
    # Processing
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='PENDING')
    reviewed_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='reviewed_applications')
    review_date = models.DateTimeField(null=True)
    review_remarks = models.TextField(blank=True)
    
    # Disbursement
    amount_approved = models.DecimalField(max_digits=10, decimal_places=2, null=True)
    disbursement_date = models.DateTimeField(null=True)
    transaction_reference = models.CharField(max_length=100, blank=True)
    
    class Meta:
        unique_together = ['student', 'scholarship_type', 'academic_year', 'semester']

class Award(models.Model):
    """Academic and Non-academic Awards"""
    AWARD_CATEGORY = [
        ('ACADEMIC', 'Academic Excellence'),
        ('RESEARCH', 'Research'),
        ('SPORTS', 'Sports'),
        ('CULTURAL', 'Cultural'),
        ('INNOVATION', 'Innovation'),
        ('LEADERSHIP', 'Leadership'),
        ('COMMUNITY', 'Community Service'),
    ]
    
    name = models.CharField(max_length=200)
    category = models.CharField(max_length=20, choices=AWARD_CATEGORY)
    description = models.TextField()
    criteria = models.TextField()
    prize_amount = models.DecimalField(max_digits=10, decimal_places=2, null=True)
    certificate_provided = models.BooleanField(default=True)
    applicable_programmes = models.ManyToManyField(Programme, blank=True)
    is_active = models.BooleanField(default=True)

class AwardRecipient(models.Model):
    """Students who received awards"""
    award = models.ForeignKey(Award, on_delete=models.CASCADE)
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    academic_year = models.CharField(max_length=9)
    award_date = models.DateField()
    citation = models.TextField(blank=True)
    certificate_issued = models.BooleanField(default=False)
    awarded_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    
    class Meta:
        unique_together = ['award', 'student', 'academic_year']

class MeritList(models.Model):
    """Batch-wise merit list"""
    batch = models.ForeignKey(Batch, on_delete=models.CASCADE)
    academic_year = models.CharField(max_length=9)
    semester = models.IntegerField()
    generated_date = models.DateTimeField(auto_now_add=True)
    
class MeritListEntry(models.Model):
    """Individual entries in merit list"""
    merit_list = models.ForeignKey(MeritList, on_delete=models.CASCADE)
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    rank = models.IntegerField()
```

---

## API Endpoints Required

### From Core Modules

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/programme_curriculum/api/programmes/` | GET | Get all programmes |
| `/programme_curriculum/api/batches/` | GET | Get all batches |
| `/programme_curriculum/api/disciplines/` | GET | Get all disciplines |
| `/academic-procedures/api/students/` | GET | Get student details |

### New Endpoints to Create

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/scholarships/api/types/` | GET, POST | Scholarship types |
| `/scholarships/api/applications/` | GET, POST | Applications |
| `/scholarships/api/applications/<id>/approve/` | POST | Approve application |
| `/scholarships/api/awards/` | GET, POST | Awards management |
| `/scholarships/api/merit-list/<batch_id>/` | GET | Get merit list |
| `/scholarships/api/eligible-students/<scholarship_id>/` | GET | Get eligible students |

---

## Integration Functions

### Validate Scholarship Eligibility

```python
def check_scholarship_eligibility(student, scholarship_type):
    """
    Check if student is eligible for a scholarship
    Returns: (is_eligible: bool, reasons: list)
    """
    reasons = []
    
    # Check backlogs
    backlog_count = course_registration.objects.filter(
        student_id=student,
        registration_type='Backlog'
    ).count()
    
    if backlog_count > scholarship_type.max_backlogs:
        reasons.append(f"Has {backlog_count} backlogs (max allowed: {scholarship_type.max_backlogs})")
    
    # Check category
    if scholarship_type.applicable_categories:
        valid_categories = scholarship_type.applicable_categories.split(',')
        if student.category not in valid_categories:
            reasons.append(f"Category {student.category} not eligible")
    
    # Check programme
    if scholarship_type.applicable_programmes.exists():
        student_programme = Programme.objects.filter(
            name__icontains=student.programme
        ).first()
        if student_programme not in scholarship_type.applicable_programmes.all():
            reasons.append("Programme not eligible")
    
    # Check dues
    try:
        dues = Dues.objects.get(student_id=student)
        if dues.academic_due > 0:
            reasons.append("Has pending academic dues")
    except Dues.DoesNotExist:
        pass
    
    return (len(reasons) == 0, reasons)
```

### Generate Merit List

```python
def generate_merit_list(batch_id, academic_year, semester):
    """Generate merit list for a batch"""
    from applications.academic_information.models import Student
    from applications.programme_curriculum.models import Batch
    
    batch = Batch.objects.get(id=batch_id)
    
    # Get all students in batch
    students = Student.objects.filter(
        batch_id=batch,
    )
    
    # Create merit list
    merit_list = MeritList.objects.create(
        batch=batch,
        academic_year=academic_year,
        semester=semester
    )
    
    # Create entries
    for rank, student in enumerate(students, 1):
        MeritListEntry.objects.create(
            merit_list=merit_list,
            student=student,
            rank=rank
        )
    
    return merit_list
```

### Auto-populate Application Data

```python
def create_scholarship_application(student_id, scholarship_type_id, academic_year, semester):
    """Create scholarship application with auto-populated data"""
    student = Student.objects.get(id=student_id)
    scholarship_type = ScholarshipType.objects.get(id=scholarship_type_id)
    
    # Check eligibility first
    is_eligible, reasons = check_scholarship_eligibility(student, scholarship_type)
    
    if not is_eligible:
        return None, reasons
    
    # Create application
    application = ScholarshipApplication.objects.create(
        student=student,
        scholarship_type=scholarship_type,
        academic_year=academic_year,
        semester=semester,
        category_at_application=student.category
    )
    
    return application, []
```

---

## Important Sync Points

1. **Registration Type**: Monitor `academic_procedures.course_registration` for backlog tracking.

3. **Fee Payment Status**: Check `academic_procedures.FeePayments` before disbursement.

4. **Dues Clearance**: Verify no pending dues in `academic_procedures.Dues` before approval.

5. **Batch Status**: Only consider `running_batch=True` from `programme_curriculum.Batch`.

---

## Notifications Integration

```python
from notification.views import scholarships_notif

# Notify student on application status change
def notify_application_status(application):
    scholarships_notif(
        sender=application.reviewed_by.user,
        recipient=application.student.id.user,
        type=f'scholarship_{application.status.lower()}'
    )
```

---

