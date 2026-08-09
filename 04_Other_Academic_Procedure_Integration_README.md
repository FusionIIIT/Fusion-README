# Other Academic Procedure Module - Integration Guide

**What this module is:** The Other Academic Procedure module will handle miscellaneous academic requests that don't fit into core registration - bonafide certificates, thesis topic approval, assistantship claims, branch change applications, no-dues certificates, and special permissions.

**Why we need to build this:** Beyond course registration, students need various academic services. Currently bonafide certificates require paper applications, thesis approvals are tracked manually. This module will automate these workflows - digital applications, automatic verification, and faster processing.

**Why integration is needed:** Every request requires student verification. Branch change needs to check current batch, target batch availability, and semester. Thesis approval needs supervisor (Faculty) information. Assistantship needs to verify student status. No-dues needs to check Dues table. All data comes from core modules.

**Key dependencies:** Student, Batch, Faculty, Dues, BranchChange

---

## Module Overview
The Other Academic Procedure module handles academic processes not covered by core registration, including bonafide certificates, thesis management, assistantship claims, branch changes, dues management, and special academic requests.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Programme`
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `id` | AutoField | Primary Key | Programme reference |
| `category` | CharField(3) | UG/PG/PHD | Process eligibility |
| `name` | CharField(70) | Programme name | Certificate generation |

#### Table: `Batch`
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `id` | AutoField | Primary Key | Batch reference |
| `name` | CharField(50) | Batch name | Certificate content |
| `discipline` | ForeignKey(Discipline) | Discipline | Branch change options |
| `year` | PositiveIntegerField | Batch year | Year-based processing |
| `curriculum` | ForeignKey(Curriculum) | Curriculum | Academic tracking |
| `running_batch` | BooleanField | Is active | Active batch filter |

#### Table: `Discipline`
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `id` | AutoField | Primary Key | Discipline reference |
| `name` | CharField(100) | Discipline name | Branch change target |
| `acronym` | CharField(10) | Short code | Display purposes |

#### Table: `Semester`
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `id` | AutoField | Primary Key | Semester reference |
| `curriculum` | ForeignKey(Curriculum) | Curriculum | Curriculum context |
| `semester_no` | PositiveIntegerField | Semester number | Semester identification |

#### Table: `Course`
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `id` | AutoField | Primary Key | Course reference |
| `code` | CharField(10) | Course code | Teaching credit courses |
| `name` | CharField(100) | Course name | Display purposes |
| `credit` | PositiveIntegerField | Credits | Credit calculation |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Primary student reference** |
| `programme` | CharField(10) | Programme | Certificate content |
| `batch` | IntegerField | Batch year | Eligibility checking |
| `batch_id` | ForeignKey(Batch) | Batch reference | Batch information |
| `category` | CharField(10) | GEN/SC/ST/OBC | Category processing |
| `specialization` | CharField(40) | Specialization | M.Tech processes |
| `curr_semester_no` | IntegerField | Current semester | Semester tracking |

#### Table: `Curriculum` (from academic_information)
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `curriculum_id` | AutoField | Primary Key | Curriculum reference |
| `course_code` | CharField(20) | Course code | Course identification |
| `programme` | CharField(10) | Programme | Programme filtering |
| `batch` | IntegerField | Batch | Batch filtering |
| `sem` | IntegerField | Semester | Semester filtering |

---

### 3. From `academic_procedures` Module (Existing Models)

#### Table: `BranchChange`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `c_id` | AutoField | Primary Key | Branch change ID |
| `branches` | ForeignKey(DepartmentInfo) | Target branch | Requested branch |
| `user` | ForeignKey(Student) | Student | Applicant |
| `applied_date` | DateField | Application date | Timeline tracking |

#### Table: `Thesis`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `reg_id` | ForeignKey(ExtraInfo) | Registration | Student registration |
| `student_id` | ForeignKey(Student) | Student | Thesis student |
| `supervisor_id` | ForeignKey(Faculty) | Supervisor | Thesis supervisor |
| `topic` | CharField(1000) | Topic | Thesis topic |

#### Table: `ThesisTopicProcess`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `student_id` | ForeignKey(Student) | Student | Thesis applicant |
| `research_area` | CharField(50) | Research area | Research domain |
| `thesis_topic` | CharField(1000) | Topic | Proposed topic |
| `supervisor_id` | ForeignKey(Faculty) | Supervisor | Primary supervisor |
| `co_supervisor_id` | ForeignKey(Faculty) | Co-supervisor | Secondary supervisor |
| `member1/2/3` | ForeignKey(Faculty) | Committee | Committee members |
| `pending_supervisor` | BooleanField | Pending | Supervisor approval pending |
| `approval_supervisor` | BooleanField | Approved | Supervisor approved |
| `forwarded_to_hod` | BooleanField | Forwarded | Sent to HOD |
| `pending_hod` | BooleanField | Pending HOD | HOD approval pending |
| `approval_by_hod` | BooleanField | HOD approved | HOD approved |

#### Table: `Bonafide`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `student_id` | ForeignKey(Student) | Student | Applicant |
| `student_name` | CharField(50) | Name | Certificate name |
| `purpose` | CharField(100) | Purpose | Certificate purpose |
| `academic_year` | CharField(15) | Academic year | Year of study |
| `enrolled_course` | CharField(10) | Programme | Enrolled programme |
| `complaint_date` | DateTimeField | Request date | Application date |

#### Table: `AssistantshipClaim`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `student` | ForeignKey(Student) | Student | Claimant |
| `month` | CharField(10) | Month | Claim month |
| `year` | IntegerField | Year | Claim year |
| `bank_account` | CharField(11) | Account | Bank account |
| `applicability` | CharField(5) | GATE/NET/CEED | Eligibility basis |
| `ta_supervisor_remark` | BooleanField | TA approval | TA supervisor approved |
| `ta_supervisor` | ForeignKey(Faculty) | TA supervisor | TA supervisor |
| `thesis_supervisor_remark` | BooleanField | Thesis approval | Thesis supervisor approved |
| `thesis_supervisor` | ForeignKey(Faculty) | Thesis supervisor | Research supervisor |
| `hod_approval` | BooleanField | HOD approved | HOD approval |
| `acad_approval` | BooleanField | Acad approved | Academic section approval |
| `account_approval` | BooleanField | Account approved | Accounts approval |
| `stipend` | IntegerField | Amount | Stipend amount |

#### Table: `Dues`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `student_id` | ForeignKey(Student) | Student | Student reference |
| `mess_due` | IntegerField | Mess dues | Mess pending |
| `hostel_due` | IntegerField | Hostel dues | Hostel pending |
| `library_due` | IntegerField | Library dues | Library pending |
| `placement_cell_due` | IntegerField | Placement dues | Placement pending |
| `academic_due` | IntegerField | Academic dues | Academic pending |

#### Table: `MessDue`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `student` | ForeignKey(Student) | Student | Student reference |
| `month` | CharField(10) | Month | Due month |
| `year` | IntegerField | Year | Due year |
| `description` | CharField(15) | Status | Paid/Due |
| `amount` | IntegerField | Amount | Due amount |
| `remaining_amount` | IntegerField | Remaining | Pending amount |

#### Table: `TeachingCreditRegistration`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `student_id` | ForeignKey(Student) | Student | Student reference |
| `curr_1/2/3/4` | ForeignKey(Curriculum) | Preferences | Course preferences |
| `req_pending` | BooleanField | Pending | Request pending |
| `approved_course` | ForeignKey(Curriculum) | Approved | Approved course |
| `course_completion` | BooleanField | Completed | Course completed |
| `supervisor_id` | ForeignKey(Faculty) | Supervisor | Supervising faculty |

#### Table: `MTechGraduateSeminarReport`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `student` | ForeignKey(Student) | Student | Presenting student |
| `theme_of_work` | TextField | Theme | Research theme |
| `date` | DateField | Date | Seminar date |
| `quality_of_work` | CharField(20) | Quality rating | Work quality |
| `quantity_of_work` | CharField(15) | Quantity rating | Work quantity |
| `Overall_grade` | CharField(2) | Grade | Overall grade |
| `panel_report` | CharField(15) | Recommendation | Panel decision |

#### Table: `PhDProgressExamination`
| Column | Type | Description | Purpose |
|--------|------|-------------|---------|
| `student` | ForeignKey(Student) | Student | PhD student |
| `theme` | CharField(50) | Theme | Research theme |
| `seminar_date_time` | DateTimeField | Date/Time | Examination datetime |
| `quality_of_work` | CharField(20) | Quality | Work quality |
| `quantity_of_work` | CharField(15) | Quantity | Work quantity |
| `Overall_grade` | CharField(2) | Grade | Overall grade |

---

### 4. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `id` | CharField(20) | User ID | User identification |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | Role identification |
| `department` | ForeignKey(DepartmentInfo) | Department | Department reference |

#### Table: `Faculty`
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | Faculty reference for supervision |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in Other Academic |
|--------|------|-------------|------------------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | Branch change target |

---

## Integration Functions

### Branch Change Process
```python
def check_branch_change_eligibility(student):
    """
    Check if student is eligible for branch change
    Rules: Only 1st year B.Tech
    """
    from applications.academic_information.models import Student
    from applications.programme_curriculum.models import Batch
    
    # Check programme
    if student.programme.upper() != 'B.TECH':
        return False, "Only B.Tech students eligible"
    
    # Check semester (only after 2nd semester)
    if student.curr_semester_no != 2:
        return False, "Only 2nd semester students eligible"
    
    return True, "Eligible for branch change"

def get_available_branches(student):
    """Get branches available for transfer"""
    from applications.globals.models import DepartmentInfo
    from applications.programme_curriculum.models import Discipline, Batch
    
    current_discipline = student.batch_id.discipline
    
    # Get all disciplines except current
    available = Discipline.objects.exclude(id=current_discipline.id)
    
    # Check seat availability
    result = []
    for disc in available:
        batch = Batch.objects.filter(
            discipline=disc,
            year=student.batch,
            running_batch=True
        ).first()
        if batch and batch.total_seats > 0:
            result.append({
                'discipline': disc,
                'available_seats': batch.total_seats,  # Need to subtract filled
            })
    
    return result
```

### Thesis Management
```python
def create_thesis_request(student_id, topic, research_area, supervisor_id, co_supervisor_id=None):
    """Create thesis topic request"""
    from applications.academic_procedures.models import ThesisTopicProcess
    from applications.academic_information.models import Student
    from applications.globals.models import Faculty
    
    student = Student.objects.get(id=student_id)
    supervisor = Faculty.objects.get(id=supervisor_id)
    
    thesis = ThesisTopicProcess.objects.create(
        student_id=student,
        thesis_topic=topic,
        research_area=research_area,
        supervisor_id=supervisor,
        co_supervisor_id=Faculty.objects.get(id=co_supervisor_id) if co_supervisor_id else None,
        pending_supervisor=True,
        submission_by_student=True
    )
    
    return thesis

def approve_thesis_by_supervisor(thesis_id, approve=True, members=None):
    """Supervisor approves/rejects thesis"""
    from applications.academic_procedures.models import ThesisTopicProcess
    
    thesis = ThesisTopicProcess.objects.get(id=thesis_id)
    
    if approve:
        thesis.pending_supervisor = False
        thesis.approval_supervisor = True
        thesis.forwarded_to_hod = True
        thesis.pending_hod = True
        
        if members:
            thesis.member1 = members.get('member1')
            thesis.member2 = members.get('member2')
            thesis.member3 = members.get('member3')
        
        thesis.save()
        return True, "Thesis approved and forwarded to HOD"
    else:
        thesis.pending_supervisor = False
        thesis.approval_supervisor = False
        thesis.save()
        return True, "Thesis rejected"
```

### Assistantship Claim Processing
```python
def create_assistantship_claim(student_id, month, year, bank_account, applicability, 
                               ta_supervisor_id, thesis_supervisor_id):
    """Create assistantship claim"""
    from applications.academic_procedures.models import AssistantshipClaim
    from applications.academic_information.models import Student
    from applications.globals.models import Faculty
    
    student = Student.objects.get(id=student_id)
    
    claim = AssistantshipClaim.objects.create(
        student=student,
        month=month,
        year=year,
        bank_account=bank_account,
        applicability=applicability,
        ta_supervisor=Faculty.objects.get(id=ta_supervisor_id),
        thesis_supervisor=Faculty.objects.get(id=thesis_supervisor_id),
    )
    
    return claim

def process_assistantship_approval(claim_id, approver_type, approve=True, stipend=None):
    """Process approval at different stages"""
    from applications.academic_procedures.models import AssistantshipClaim
    
    claim = AssistantshipClaim.objects.get(id=claim_id)
    
    if approver_type == 'ta_supervisor':
        claim.ta_supervisor_remark = approve
    elif approver_type == 'thesis_supervisor':
        claim.thesis_supervisor_remark = approve
    elif approver_type == 'hod':
        claim.hod_approval = approve
    elif approver_type == 'acad':
        claim.acad_approval = approve
    elif approver_type == 'account':
        claim.account_approval = approve
        if approve and stipend:
            claim.stipend = stipend
    
    claim.save()
    return claim
```

### Bonafide Certificate Generation
```python
def generate_bonafide(student_id, purpose):
    """Generate bonafide certificate request"""
    from applications.academic_procedures.models import Bonafide
    from applications.academic_information.models import Student
    from django.utils import timezone
    
    student = Student.objects.select_related('id', 'id__user').get(id=student_id)
    
    # Get academic year
    current_date = timezone.now()
    if current_date.month >= 7:
        academic_year = f"{current_date.year}-{str(current_date.year + 1)[-2:]}"
    else:
        academic_year = f"{current_date.year - 1}-{str(current_date.year)[-2:]}"
    
    bonafide = Bonafide.objects.create(
        student_id=student,
        student_name=f"{student.id.user.first_name} {student.id.user.last_name}",
        purpose=purpose,
        academic_year=academic_year,
        enrolled_course=student.programme,
    )
    
    return bonafide
```

### Dues Management
```python
def get_student_dues(student_id):
    """Get all dues for a student"""
    from applications.academic_procedures.models import Dues, MessDue
    from applications.academic_information.models import Student
    
    student = Student.objects.get(id=student_id)
    
    # Get consolidated dues
    try:
        dues = Dues.objects.get(student_id=student)
        total_dues = (
            dues.mess_due + 
            dues.hostel_due + 
            dues.library_due + 
            dues.placement_cell_due + 
            dues.academic_due
        )
    except Dues.DoesNotExist:
        dues = None
        total_dues = 0
    
    # Get mess dues detail
    mess_dues = MessDue.objects.filter(
        student=student,
        description='Stu_due'
    ).order_by('-year', '-month')
    
    return {
        'consolidated_dues': dues,
        'total_dues': total_dues,
        'mess_dues_detail': list(mess_dues),
        'no_dues_status': total_dues == 0
    }

def check_no_dues(student_id):
    """Check if student has no pending dues"""
    dues_info = get_student_dues(student_id)
    return dues_info['no_dues_status']
```

---

## New Models to Extend

```python
from django.db import models
from applications.academic_information.models import Student
from applications.globals.models import ExtraInfo, Faculty

class NoDuesCertificate(models.Model):
    """No Dues Certificate tracking"""
    STATUS_CHOICES = [
        ('PENDING', 'Pending'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
    ]
    
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    applied_date = models.DateTimeField(auto_now_add=True)
    
    # Department-wise clearance
    library_cleared = models.BooleanField(default=False)
    hostel_cleared = models.BooleanField(default=False)
    mess_cleared = models.BooleanField(default=False)
    accounts_cleared = models.BooleanField(default=False)
    department_cleared = models.BooleanField(default=False)
    
    # Final status
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='PENDING')
    certificate_number = models.CharField(max_length=50, blank=True)
    issue_date = models.DateTimeField(null=True, blank=True)
    
class CharacterCertificate(models.Model):
    """Character Certificate Request"""
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    purpose = models.CharField(max_length=200)
    applied_date = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=20, default='PENDING')
    verified_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    issue_date = models.DateTimeField(null=True, blank=True)
    certificate_number = models.CharField(max_length=50, blank=True)

class MigrationCertificate(models.Model):
    """Migration Certificate for students leaving"""
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    reason = models.CharField(max_length=200)
    destination_institute = models.CharField(max_length=200)
    applied_date = models.DateTimeField(auto_now_add=True)
    no_dues_cleared = models.BooleanField(default=False)
    status = models.CharField(max_length=20, default='PENDING')
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    issue_date = models.DateTimeField(null=True, blank=True)

class TranscriptRequest(models.Model):
    """Official Transcript Request"""
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    purpose = models.CharField(max_length=200)
    copies_required = models.IntegerField(default=1)
    delivery_address = models.TextField(blank=True)
    applied_date = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=20, default='PENDING')
    fee_paid = models.BooleanField(default=False)
    transaction_id = models.CharField(max_length=100, blank=True)
    prepared_date = models.DateTimeField(null=True, blank=True)
    dispatched_date = models.DateTimeField(null=True, blank=True)

class SemesterLeave(models.Model):
    """Semester Leave (Gap Year) Request"""
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    leave_semester = models.IntegerField()
    leave_year = models.IntegerField()
    reason = models.TextField()
    supporting_documents = models.FileField(upload_to='academic_procedures/semester_leave/', null=True)
    applied_date = models.DateTimeField(auto_now_add=True)
    
    hod_approval = models.BooleanField(null=True)
    hod_remarks = models.TextField(blank=True)
    dean_approval = models.BooleanField(null=True)
    dean_remarks = models.TextField(blank=True)
    
    status = models.CharField(max_length=20, default='PENDING')
```

---

## API Endpoints

### Existing Endpoints (from `academic_procedures/urls.py`)
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/academic-procedures/brach-change-request/` | POST | Branch change request |
| `/academic-procedures/branch-change/` | POST | Approve branch change |
| `/academic-procedures/addThesis/` | POST | Add thesis |
| `/academic-procedures/ACF/` | POST | Assistantship claim form |
| `/academic-procedures/bonafide_pdf/` | POST | Generate bonafide |
| `/academic-procedures/dues_pdf/` | GET | Dues report |

### New Endpoints to Create
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/other-academic/api/no-dues/apply/` | POST | Apply for no dues |
| `/other-academic/api/no-dues/status/<id>/` | GET | Check no dues status |
| `/other-academic/api/character-certificate/` | POST | Request character cert |
| `/other-academic/api/transcript/` | POST | Request transcript |
| `/other-academic/api/semester-leave/` | POST | Apply for semester leave |
| `/other-academic/api/migration/` | POST | Apply for migration |

---

## Workflow Integration

### Branch Change Workflow
```
1. Student applies (BranchChange table)
2. Admin verifies eligibility
3. Check seat availability in target branch
4. HOD approval (both departments)
5. Dean approval
6. Update Student.batch_id to new batch
```

### Thesis Approval Workflow
```
1. Student submits (ThesisTopicProcess)
2. Supervisor reviews (approval_supervisor)
3. Forward to HOD (forwarded_to_hod)
4. HOD approval (approval_by_hod)
5. Committee formation (member1, member2, member3)
```

### Assistantship Workflow
```
1. Student claims (AssistantshipClaim)
2. TA Supervisor approval (ta_supervisor_remark)
3. Thesis Supervisor approval (thesis_supervisor_remark)
4. HOD approval (hod_approval)
5. Academic section approval (acad_approval)
6. Accounts approval & disbursement (account_approval)
```

---

## Important Notes

1. **Multi-stage Approvals**: Most processes require multi-level approvals (supervisor → HOD → Dean/Admin).

3. **Document Generation**: Bonafide, No Dues, etc. require PDF generation capabilities.

4. **Dues Verification**: Many processes require dues clearance before approval.

5. **Notification Integration**: Use `notification.views.academics_module_notif` for status updates.

---

