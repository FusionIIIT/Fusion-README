# Placement Cell Module - Integration Guide

**What this module is:** The Placement Cell module will manage campus recruitment - company registrations, job postings, student placement profiles, application tracking, interview scheduling, offer management, and placement statistics.

**Why we need to build this:** Campus placements are crucial for students but currently the process is fragmented - companies register via email, students submit profiles on paper, eligibility is checked manually. This module will digitize the entire placement process, automating eligibility checks against academic records.

**Why integration is needed:** Placement must verify student identity (from Student), check batch/discipline (from programme_curriculum), verify backlog status (from Student_grades), and get contact details (from ExtraInfo). Without production data, eligibility cannot be automated.

**Key dependencies:** Student, Batch, Discipline, Programme, Student_grades, ExtraInfo

---

## Module Overview
The Placement Cell module manages campus recruitment activities, company registrations, job postings, student profiles, application tracking, interview scheduling, and placement statistics.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Programme`
| Column | Type | Description | Usage in Placement |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Eligible programmes |
| `name` | CharField(70) | Programme name | **Programme filtering** |
| `category` | CharField(20) | UG/PG/PhD | **Eligibility criteria** |

#### Table: `Discipline`
| Column | Type | Description | Usage in Placement |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Branch reference |
| `name` | CharField(100) | Discipline name | **Branch eligibility** |
| `acronym` | CharField(10) | Short code | Quick reference |

#### Table: `Batch`
| Column | Type | Description | Usage in Placement |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Batch reference |
| `name` | CharField(50) | Batch name | **Passing out batch** |
| `year` | IntegerField | Year | **Batch year** |
| `discipline` | ForeignKey(Discipline) | Discipline | Programme context |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in Placement |
|--------|------|-------------|-------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Student for placement** |
| `batch_id` | ForeignKey(Batch) | Batch | **Eligible batch** |
| `category` | CharField(20) | Category | Category reservation |
| `father_name` | CharField(40) | Father name | Profile |
| `mother_name` | CharField(40) | Mother name | Profile |

#### Table: `Student_grades` (from online_cms)
| Column | Type | Description | Usage in Placement |
|--------|------|-------------|-------------------|
| `roll_no` | TextField | Student roll number | **Academic record** |
| `grade` | TextField | Grade | Performance |

---

### 3. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Placement |
|--------|------|-------------|-------------------|
| `id` | CharField(20) | User ID (Roll No) | **Student ID** |
| `user` | OneToOneField(User) | User | User reference |
| `sex` | CharField(2) | Gender | Demographics |
| `date_of_birth` | DateField | DOB | **Age verification** |
| `phone_no` | BigIntegerField | Phone | **Contact** |
| `address` | TextField | Address | Permanent address |
| `profile_picture` | ImageField | Photo | Profile |

#### Table: `Faculty`
| Column | Type | Description | Usage in Placement |
|--------|------|-------------|-------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **TPO/Coordinator** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in Placement |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | Coordinator dept |

---

## Data Models to Create in Placement Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, Faculty, DepartmentInfo
from applications.academic_information.models import Student
from applications.programme_curriculum.models import Programme, Discipline, Batch

# ==================== COMPANY MANAGEMENT ====================

class Company(models.Model):
    """Registered companies"""
    COMPANY_TYPE = [
        ('IT', 'IT/Software'),
        ('CORE', 'Core Engineering'),
        ('FINANCE', 'Finance/Banking'),
        ('CONSULTING', 'Consulting'),
        ('MANUFACTURING', 'Manufacturing'),
        ('STARTUP', 'Startup'),
        ('PSU', 'PSU'),
        ('RESEARCH', 'Research Organization'),
        ('OTHER', 'Other'),
    ]
    
    name = models.CharField(max_length=200)
    company_type = models.CharField(max_length=20, choices=COMPANY_TYPE)
    
    # Details
    website = models.URLField(blank=True)
    about = models.TextField(blank=True)
    
    # Location
    headquarters = models.CharField(max_length=200, blank=True)
    
    # HR Contact
    hr_name = models.CharField(max_length=100, blank=True)
    hr_email = models.EmailField(blank=True)
    hr_phone = models.CharField(max_length=15, blank=True)
    
    # Status
    is_active = models.BooleanField(default=True)
    registration_date = models.DateField(auto_now_add=True)
    
    class Meta:
        verbose_name_plural = "Companies"
        ordering = ['name']

# ==================== PLACEMENT SEASON ====================

class PlacementSeason(models.Model):
    """Placement seasons/years"""
    SEASON_TYPE = [
        ('PLACEMENT', 'Final Placement'),
        ('INTERNSHIP', 'Internship'),
        ('PPO', 'Pre-Placement Offer'),
    ]
    
    name = models.CharField(max_length=50)  # e.g., "2024-25"
    season_type = models.CharField(max_length=20, choices=SEASON_TYPE)
    
    # Target batch
    batch = models.ForeignKey(Batch, on_delete=models.CASCADE)
    
    # Timeline
    start_date = models.DateField()
    end_date = models.DateField()
    
    # Registration
    registration_start = models.DateField()
    registration_end = models.DateField()
    
    is_active = models.BooleanField(default=False)

# ==================== JOB POSTINGS ====================

class JobPosting(models.Model):
    """Job/Internship postings"""
    STATUS = [
        ('DRAFT', 'Draft'),
        ('OPEN', 'Open'),
        ('CLOSED', 'Closed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    JOB_TYPE = [
        ('PLACEMENT', 'Full-Time'),
        ('INTERNSHIP', 'Internship'),
        ('6M_INTERN', '6-Month Internship'),
    ]
    
    company = models.ForeignKey(Company, on_delete=models.CASCADE, related_name='job_postings')
    season = models.ForeignKey(PlacementSeason, on_delete=models.CASCADE)
    
    # Job details
    title = models.CharField(max_length=200)
    job_type = models.CharField(max_length=20, choices=JOB_TYPE)
    description = models.TextField()
    job_location = models.CharField(max_length=200, blank=True)
    
    # Package
    ctc = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)  # In LPA
    ctc_breakup = models.TextField(blank=True)
    stipend = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)  # For internship
    
    # Eligibility
    eligible_programmes = models.ManyToManyField(Programme, blank=True)
    eligible_disciplines = models.ManyToManyField(Discipline, blank=True)
    backlog_allowed = models.BooleanField(default=False)
    
    # Skills required
    skills_required = models.TextField(blank=True)
    
    # Bond
    has_bond = models.BooleanField(default=False)
    bond_details = models.TextField(blank=True)
    
    # Timeline
    application_deadline = models.DateTimeField()
    
    # Documents
    job_description_file = models.FileField(upload_to='placement/jd/', blank=True)
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='DRAFT')
    
    created_at = models.DateTimeField(auto_now_add=True)
    created_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    
    # Openings
    total_openings = models.IntegerField(null=True, blank=True)

# ==================== STUDENT PROFILE ====================

class StudentPlacementProfile(models.Model):
    """Extended placement profile for students"""
    student = models.OneToOneField(Student, on_delete=models.CASCADE, related_name='placement_profile')
    
    # Academic
    tenth_percentage = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)
    tenth_board = models.CharField(max_length=100, blank=True)
    tenth_year = models.IntegerField(null=True, blank=True)
    
    twelfth_percentage = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)
    twelfth_board = models.CharField(max_length=100, blank=True)
    twelfth_year = models.IntegerField(null=True, blank=True)
    
    # Backlogs
    active_backlogs = models.IntegerField(default=0)
    total_backlogs = models.IntegerField(default=0)
    
    # Skills
    skills = models.TextField(blank=True)  # JSON list
    programming_languages = models.TextField(blank=True)
    
    # Experience
    internship_experience = models.TextField(blank=True)
    project_experience = models.TextField(blank=True)
    
    # Documents
    resume = models.FileField(upload_to='placement/resumes/', blank=True)
    
    # LinkedIn
    linkedin_url = models.URLField(blank=True)
    github_url = models.URLField(blank=True)
    portfolio_url = models.URLField(blank=True)
    
    # Preferences
    preferred_locations = models.TextField(blank=True)
    preferred_sectors = models.TextField(blank=True)
    
    # Status
    is_registered = models.BooleanField(default=False)
    is_placed = models.BooleanField(default=False)
    opted_out = models.BooleanField(default=False)
    opted_out_reason = models.TextField(blank=True)
    
    updated_at = models.DateTimeField(auto_now=True)

# ==================== APPLICATION ====================

class JobApplication(models.Model):
    """Student job applications"""
    STATUS = [
        ('APPLIED', 'Applied'),
        ('SHORTLISTED', 'Shortlisted'),
        ('TEST_SCHEDULED', 'Test Scheduled'),
        ('TEST_CLEARED', 'Test Cleared'),
        ('INTERVIEW_SCHEDULED', 'Interview Scheduled'),
        ('SELECTED', 'Selected'),
        ('REJECTED', 'Rejected'),
        ('WITHDRAWN', 'Withdrawn'),
        ('OFFER_RECEIVED', 'Offer Received'),
        ('OFFER_ACCEPTED', 'Offer Accepted'),
        ('OFFER_DECLINED', 'Offer Declined'),
    ]
    
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='job_applications')
    job = models.ForeignKey(JobPosting, on_delete=models.CASCADE, related_name='applications')
    
    applied_at = models.DateTimeField(auto_now_add=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='APPLIED')
    status_updated_at = models.DateTimeField(auto_now=True)
    
    remarks = models.TextField(blank=True)
    
    class Meta:
        unique_together = ['student', 'job']

class ApplicationStatusHistory(models.Model):
    """Track application status changes"""
    application = models.ForeignKey(JobApplication, on_delete=models.CASCADE, related_name='status_history')
    status = models.CharField(max_length=20)
    changed_at = models.DateTimeField(auto_now_add=True)
    changed_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    remarks = models.TextField(blank=True)

# ==================== INTERVIEW SCHEDULING ====================

class InterviewRound(models.Model):
    """Interview rounds for a job"""
    ROUND_TYPE = [
        ('APTITUDE', 'Aptitude Test'),
        ('CODING', 'Coding Test'),
        ('TECHNICAL', 'Technical Interview'),
        ('HR', 'HR Interview'),
        ('GROUP_DISCUSSION', 'Group Discussion'),
        ('CASE_STUDY', 'Case Study'),
        ('FINAL', 'Final Round'),
    ]
    
    job = models.ForeignKey(JobPosting, on_delete=models.CASCADE, related_name='interview_rounds')
    round_number = models.IntegerField()
    round_type = models.CharField(max_length=20, choices=ROUND_TYPE)
    round_name = models.CharField(max_length=100, blank=True)
    
    scheduled_date = models.DateField(null=True, blank=True)
    scheduled_time = models.TimeField(null=True, blank=True)
    venue = models.CharField(max_length=200, blank=True)
    
    # Online test
    is_online = models.BooleanField(default=False)
    test_link = models.URLField(blank=True)
    
    duration_minutes = models.IntegerField(null=True, blank=True)
    
    instructions = models.TextField(blank=True)
    
    class Meta:
        unique_together = ['job', 'round_number']
        ordering = ['job', 'round_number']

class InterviewSchedule(models.Model):
    """Individual interview schedules"""
    application = models.ForeignKey(JobApplication, on_delete=models.CASCADE, related_name='interviews')
    round = models.ForeignKey(InterviewRound, on_delete=models.CASCADE)
    
    scheduled_datetime = models.DateTimeField()
    venue = models.CharField(max_length=200, blank=True)
    
    # Result
    attended = models.BooleanField(default=False)
    cleared = models.BooleanField(null=True)
    remarks = models.TextField(blank=True)
    
    class Meta:
        unique_together = ['application', 'round']

# ==================== PLACEMENT OFFERS ====================

class PlacementOffer(models.Model):
    """Placement/Internship offers"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('ACCEPTED', 'Accepted'),
        ('DECLINED', 'Declined'),
        ('EXPIRED', 'Expired'),
    ]
    
    application = models.OneToOneField(JobApplication, on_delete=models.CASCADE, related_name='offer')
    
    offer_date = models.DateField()
    offer_letter = models.FileField(upload_to='placement/offers/', blank=True)
    
    offered_ctc = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    offered_stipend = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    joining_date = models.DateField(null=True, blank=True)
    offer_valid_until = models.DateField()
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    response_date = models.DateField(null=True, blank=True)
    response_remarks = models.TextField(blank=True)

# ==================== NOTIFICATIONS ====================

class PlacementNotification(models.Model):
    """Placement notifications"""
    NOTIFICATION_TYPE = [
        ('JOB', 'New Job Posting'),
        ('DEADLINE', 'Application Deadline'),
        ('SCHEDULE', 'Interview Schedule'),
        ('RESULT', 'Result Announcement'),
        ('GENERAL', 'General'),
    ]
    
    title = models.CharField(max_length=200)
    content = models.TextField()
    notification_type = models.CharField(max_length=20, choices=NOTIFICATION_TYPE)
    
    # Target
    job = models.ForeignKey(JobPosting, on_delete=models.SET_NULL, null=True, blank=True)
    batch = models.ForeignKey(Batch, on_delete=models.SET_NULL, null=True, blank=True)
    programme = models.ForeignKey(Programme, on_delete=models.SET_NULL, null=True, blank=True)
    
    posted_at = models.DateTimeField(auto_now_add=True)
    posted_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    
    is_active = models.BooleanField(default=True)

# ==================== STATISTICS ====================

class PlacementStatistics(models.Model):
    """Placement statistics per season"""
    season = models.ForeignKey(PlacementSeason, on_delete=models.CASCADE, related_name='statistics')
    discipline = models.ForeignKey(Discipline, on_delete=models.CASCADE)
    
    total_students = models.IntegerField(default=0)
    registered_students = models.IntegerField(default=0)
    placed_students = models.IntegerField(default=0)
    
    highest_package = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    average_package = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    median_package = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    
    companies_visited = models.IntegerField(default=0)
    total_offers = models.IntegerField(default=0)
    
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        unique_together = ['season', 'discipline']
```

---

## Integration Functions

### Check Student Eligibility
```python
def check_eligibility(student, job):
    """Check if student is eligible for a job posting"""
    from applications.online_cms.models import Student_grades
    
    # Get student's programme/discipline from batch
    batch = student.batch_id
    discipline = batch.discipline
    programme = discipline.programmes.first()  # Get associated programme
    
    # Check programme eligibility
    if job.eligible_programmes.exists():
        if programme not in job.eligible_programmes.all():
            return False, "Programme not eligible"
    
    # Check discipline eligibility
    if job.eligible_disciplines.exists():
        if discipline not in job.eligible_disciplines.all():
            return False, "Branch not eligible"
    
    # Check backlogs
    if not job.backlog_allowed:
        # Check for active backlogs (grades = 'F')
        backlog_count = Student_grades.objects.filter(
            roll_no=student.id.id,
            grade='F',
            verified=True
        ).count()
        if backlog_count > 0:
            return False, "Active backlogs not allowed"
    
    return True, "Eligible"
```

### Get Students by Eligibility
```python
def get_eligible_students(job):
    """Get all eligible students for a job posting"""
    from applications.academic_information.models import Student
    
    students = Student.objects.select_related('batch_id', 'batch_id__discipline', 'id')
    
    # Filter by batch
    if job.season.batch:
        students = students.filter(batch_id=job.season.batch)
    
    # Filter by discipline
    if job.eligible_disciplines.exists():
        students = students.filter(batch_id__discipline__in=job.eligible_disciplines.all())
    
    return students
```

### Get Student Placement Stats
```python
def get_student_info(student):
    """Get comprehensive student info for placement"""
    from applications.globals.models import ExtraInfo
    from applications.programme_curriculum.models import Batch
    
    extrainfo = student.id
    batch = student.batch_id
    
    return {
        'roll_no': extrainfo.id,
        'name': f"{extrainfo.user.first_name} {extrainfo.user.last_name}",
        'email': extrainfo.user.email,
        'phone': extrainfo.phone_no,
        'batch': batch.name,
        'discipline': batch.discipline.name,
        'category': student.category,
    }
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/placement/api/companies/` | GET, POST | Company management |
| `/placement/api/jobs/` | GET | Job listings |
| `/placement/api/jobs/<id>/apply/` | POST | Apply for job |
| `/placement/api/applications/` | GET | Student applications |
| `/placement/api/profile/` | GET, PUT | Placement profile |
| `/placement/api/offers/` | GET | Placement offers |
| `/placement/api/statistics/` | GET | Placement statistics |

---

## Important Sync Points

1. **Student Base**: Use `Student` model (linked to `ExtraInfo`) for all student data.
2. **Batch/Branch**: Use `Student.batch_id` → `Batch.discipline` for eligibility.
3. **Academic Record**: Check `Student_grades` from online_cms for backlog verification.
4. **Contact**: Get phone/email from `ExtraInfo` and `User`.

---

