# File Tracking System (FTS) - Integration Guide

**What this module is:** The File Tracking System will manage document workflow across the institute - tracking files as they move between offices, managing approvals, and maintaining audit trails for administrative documents.

**Why we need to build this:** In an institute, files (leave applications, purchase requests, student petitions) move through multiple offices. Currently file location is often unknown causing delays. This module will provide real-time file tracking, ensuring no file gets lost in bureaucracy.

**Why integration is needed:** Files are routed to departments (Discipline), sent by employees (ExtraInfo/Faculty/Staff), and may concern students (Student). The FTS needs to know organizational structure to route files correctly and identify senders/receivers. All identity and structure data comes from core modules.

**Key dependencies:** Discipline (departments), ExtraInfo (users), Faculty, Staff, DepartmentInfo

---

## Module Overview
The File Tracking System manages document workflow, file routing, approvals, and tracking across departments. It tracks the movement of physical and digital files through various offices and maintains audit trails.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Discipline`
| Column | Type | Description | Usage in FTS |
|--------|------|-------------|--------------|
| `id` | AutoField | Primary Key | Department routing |
| `name` | CharField(100) | Discipline name | Department identification |
| `acronym` | CharField(10) | Short code | Quick reference |

#### Table: `Proposal_Tracking` (Course Proposal FTS)
| Column | Type | Description | Usage in FTS |
|--------|------|-------------|--------------|
| `file_id` | CharField(100) | File ID | Unique file identifier |
| `current_id` | CharField(100) | Current holder | Current location |
| `current_design` | CharField(100) | Current designation | Holder designation |
| `receive_id` | ForeignKey(User) | Receiver | Next recipient |
| `receive_design` | ForeignKey(Designation) | Receiver designation | Recipient role |
| `disciplines` | ForeignKey(Discipline) | Discipline | Department context |
| `receive_date` | DateTimeField | Receipt date | Tracking timestamp |
| `forward_date` | DateTimeField | Forward date | Movement timestamp |
| `remarks` | CharField(250) | Remarks | Comments |
| `is_added` | BooleanField | Added status | Processing status |
| `is_submitted` | BooleanField | Submitted | Submission status |
| `is_rejected` | BooleanField | Rejected | Rejection status |
| `sender_archive` | BooleanField | Sender archive | Sender view |
| `receiver_archive` | BooleanField | Receiver archive | Receiver view |

#### Table: `NewProposalFile` (Course Proposals)
| Column | Type | Description | Usage in FTS |
|--------|------|-------------|--------------|
| `uploader` | CharField(100) | Uploader name | File originator |
| `designation` | CharField(100) | Uploader designation | Originator role |
| `code` | CharField(10) | Course code | Reference code |
| `name` | CharField(100) | Course name | File subject |
| `subject` | CharField(100) | Subject | File subject |
| `description` | CharField(400) | Description | File description |
| `upload_date` | DateTimeField | Upload date | Creation timestamp |
| `is_read` | BooleanField | Read status | View tracking |
| `is_update` | BooleanField | Update flag | Modification flag |
| `is_archive` | BooleanField | Archived | Archive status |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in FTS |
|--------|------|-------------|--------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | Student file requests |
| `programme` | CharField(10) | Programme | Request categorization |
| `batch_id` | ForeignKey(Batch) | Batch | Batch reference |

---

### 3. From `academic_procedures` Module

#### Files that Require Tracking:
- **Branch Change Applications** (`BranchChange`)
- **Thesis Topic Approvals** (`ThesisTopicProcess`)
- **Assistantship Claims** (`AssistantshipClaim`)
- **Bonafide Requests** (`Bonafide`)
- **Fee Payment Records** (`FeePayments`)

---

### 4. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in FTS |
|--------|------|-------------|--------------|
| `id` | CharField(20) | User ID | **File creator/handler ID** |
| `user` | OneToOneField(User) | User | User authentication |
| `user_type` | CharField(20) | Type | Role-based routing |
| `department` | ForeignKey(DepartmentInfo) | Department | **Department routing** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in FTS |
|--------|------|-------------|--------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | **Routing destination** |

#### Table: `Designation`
| Column | Type | Description | Usage in FTS |
|--------|------|-------------|--------------|
| `id` | AutoField | Primary Key | Designation reference |
| `name` | CharField(50) | Designation name | **Role-based routing** |
| `full_name` | CharField(100) | Full name | Display purposes |
| `type` | CharField(30) | Type | academic/administrative |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in FTS |
|--------|------|-------------|--------------|
| `user` | ForeignKey(User) | Permanent holder | Original holder |
| `working` | ForeignKey(User) | Current holder | **Active file handler** |
| `designation` | ForeignKey(Designation) | Designation | Role assignment |
| `held_at` | DateTimeField | Timestamp | Assignment date |

---

## Data Models to Create in FTS Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, Designation, DepartmentInfo, HoldsDesignation

class FileType(models.Model):
    """Types of files that can be tracked"""
    FILE_CATEGORIES = [
        ('ACADEMIC', 'Academic'),
        ('ADMINISTRATIVE', 'Administrative'),
        ('FINANCIAL', 'Financial'),
        ('HR', 'Human Resource'),
        ('ESTABLISHMENT', 'Establishment'),
        ('RESEARCH', 'Research'),
        ('STUDENT', 'Student Related'),
        ('OTHER', 'Other'),
    ]
    
    name = models.CharField(max_length=100)
    category = models.CharField(max_length=20, choices=FILE_CATEGORIES)
    description = models.TextField(blank=True)
    default_workflow = models.TextField(blank=True)  # JSON workflow definition
    requires_attachments = models.BooleanField(default=False)
    max_processing_days = models.IntegerField(default=7)
    is_active = models.BooleanField(default=True)

class File(models.Model):
    """Main file tracking entity"""
    FILE_STATUS = [
        ('CREATED', 'Created'),
        ('IN_PROGRESS', 'In Progress'),
        ('PENDING', 'Pending Action'),
        ('FORWARDED', 'Forwarded'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('CLOSED', 'Closed'),
        ('ARCHIVED', 'Archived'),
    ]
    
    PRIORITY_LEVELS = [
        ('LOW', 'Low'),
        ('NORMAL', 'Normal'),
        ('HIGH', 'High'),
        ('URGENT', 'Urgent'),
    ]
    
    # File identification
    file_number = models.CharField(max_length=50, unique=True)
    file_type = models.ForeignKey(FileType, on_delete=models.PROTECT)
    subject = models.CharField(max_length=500)
    description = models.TextField(blank=True)
    
    # Creation details
    created_by = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT, related_name='created_files')
    created_at = models.DateTimeField(auto_now_add=True)
    source_department = models.ForeignKey(DepartmentInfo, on_delete=models.PROTECT, related_name='outgoing_files')
    
    # Current status
    status = models.CharField(max_length=20, choices=FILE_STATUS, default='CREATED')
    priority = models.CharField(max_length=10, choices=PRIORITY_LEVELS, default='NORMAL')
    
    # Current holder
    current_holder = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT, related_name='holding_files')
    current_designation = models.ForeignKey(Designation, on_delete=models.PROTECT)
    current_department = models.ForeignKey(DepartmentInfo, on_delete=models.PROTECT, related_name='current_files')
    received_at = models.DateTimeField(auto_now=True)
    
    # Completion
    closed_at = models.DateTimeField(null=True, blank=True)
    closure_remarks = models.TextField(blank=True)
    
    # Reference to source module
    source_module = models.CharField(max_length=50, blank=True)  # e.g., 'academic_procedures'
    source_object_id = models.IntegerField(null=True, blank=True)  # ID of source object
    
    class Meta:
        ordering = ['-created_at']

class FileAttachment(models.Model):
    """Attachments to files"""
    file = models.ForeignKey(File, on_delete=models.CASCADE, related_name='attachments')
    name = models.CharField(max_length=200)
    document = models.FileField(upload_to='fts/attachments/')
    uploaded_by = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT)
    uploaded_at = models.DateTimeField(auto_now_add=True)
    description = models.TextField(blank=True)

class FileMovement(models.Model):
    """Track file movement history"""
    ACTION_TYPES = [
        ('CREATE', 'Created'),
        ('FORWARD', 'Forwarded'),
        ('RECEIVE', 'Received'),
        ('APPROVE', 'Approved'),
        ('REJECT', 'Rejected'),
        ('RETURN', 'Returned'),
        ('COMMENT', 'Comment Added'),
        ('CLOSE', 'Closed'),
        ('REOPEN', 'Reopened'),
    ]
    
    file = models.ForeignKey(File, on_delete=models.CASCADE, related_name='movements')
    action = models.CharField(max_length=20, choices=ACTION_TYPES)
    
    # Sender details
    sender = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT, related_name='sent_movements')
    sender_designation = models.ForeignKey(Designation, on_delete=models.PROTECT, related_name='sent_from_designation')
    sender_department = models.ForeignKey(DepartmentInfo, on_delete=models.PROTECT, related_name='sent_from_dept')
    
    # Receiver details
    receiver = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT, related_name='received_movements', null=True, blank=True)
    receiver_designation = models.ForeignKey(Designation, on_delete=models.PROTECT, related_name='sent_to_designation', null=True, blank=True)
    receiver_department = models.ForeignKey(DepartmentInfo, on_delete=models.PROTECT, related_name='sent_to_dept', null=True, blank=True)
    
    # Movement details
    remarks = models.TextField(blank=True)
    timestamp = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['timestamp']

class FileWorkflow(models.Model):
    """Define workflow templates"""
    name = models.CharField(max_length=100)
    file_type = models.ForeignKey(FileType, on_delete=models.CASCADE)
    step_order = models.IntegerField()
    designation = models.ForeignKey(Designation, on_delete=models.PROTECT)
    department = models.ForeignKey(DepartmentInfo, on_delete=models.PROTECT, null=True, blank=True)
    action_required = models.CharField(max_length=50)  # approve, review, sign, etc.
    max_days = models.IntegerField(default=3)
    is_mandatory = models.BooleanField(default=True)
    
    class Meta:
        ordering = ['file_type', 'step_order']
        unique_together = ['file_type', 'step_order']

class DraftFile(models.Model):
    """Draft files not yet submitted"""
    created_by = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE)
    file_type = models.ForeignKey(FileType, on_delete=models.PROTECT)
    subject = models.CharField(max_length=500)
    description = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    draft_data = models.JSONField(default=dict)  # Store draft form data
```

---

## Integration Functions

### Get User's Designations for Routing
```python
def get_user_designations(user):
    """Get all designations held by a user"""
    from applications.globals.models import HoldsDesignation
    
    return HoldsDesignation.objects.filter(
        working=user
    ).select_related('designation', 'designation__type')

def get_users_with_designation(designation_name):
    """Get all users holding a specific designation"""
    from applications.globals.models import HoldsDesignation, Designation
    
    designation = Designation.objects.filter(name__icontains=designation_name).first()
    if not designation:
        return []
    
    return HoldsDesignation.objects.filter(
        designation=designation
    ).select_related('working', 'user')
```

### File Creation from Other Modules
```python
def create_file_from_branch_change(branch_change_request):
    """Create FTS file for branch change request"""
    from applications.academic_procedures.models import BranchChange
    from applications.academic_information.models import Student
    
    student = branch_change_request.user
    student_extra = student.id
    
    # Get file type
    file_type = FileType.objects.get(name='Branch Change Request')
    
    # Generate file number
    file_number = generate_file_number('BC', student_extra.department)
    
    # Get initial handler (typically Dean Academic office)
    initial_handler = get_initial_handler_for_file_type(file_type)
    
    file = File.objects.create(
        file_number=file_number,
        file_type=file_type,
        subject=f"Branch Change Request - {student_extra.id}",
        description=f"Branch change request from {student.batch_id.discipline.name} to {branch_change_request.branches.name}",
        created_by=student_extra,
        source_department=student_extra.department,
        current_holder=initial_handler['user'],
        current_designation=initial_handler['designation'],
        current_department=initial_handler['department'],
        source_module='academic_procedures',
        source_object_id=branch_change_request.c_id,
        priority='NORMAL'
    )
    
    # Create initial movement
    FileMovement.objects.create(
        file=file,
        action='CREATE',
        sender=student_extra,
        sender_designation=Designation.objects.get(name='student'),
        sender_department=student_extra.department,
        receiver=initial_handler['user'],
        receiver_designation=initial_handler['designation'],
        receiver_department=initial_handler['department'],
        remarks='Branch change application submitted'
    )
    
    return file

def create_file_from_thesis(thesis_request):
    """Create FTS file for thesis topic approval"""
    from applications.academic_procedures.models import ThesisTopicProcess
    
    student_extra = thesis_request.student_id.id
    supervisor_extra = thesis_request.supervisor_id.id
    
    file_type = FileType.objects.get(name='Thesis Topic Approval')
    file_number = generate_file_number('TH', student_extra.department)
    
    file = File.objects.create(
        file_number=file_number,
        file_type=file_type,
        subject=f"Thesis Topic Approval - {student_extra.id}",
        description=f"Thesis: {thesis_request.thesis_topic}",
        created_by=student_extra,
        source_department=student_extra.department,
        current_holder=supervisor_extra,
        current_designation=Designation.objects.filter(name__icontains='professor').first(),
        current_department=supervisor_extra.department,
        source_module='academic_procedures',
        source_object_id=thesis_request.id,
        priority='NORMAL'
    )
    
    return file
```

### File Routing
```python
def forward_file(file_id, sender_user, receiver_designation_name, receiver_department_id=None, remarks=''):
    """Forward file to next handler"""
    from applications.globals.models import HoldsDesignation, Designation, DepartmentInfo
    
    file = File.objects.get(id=file_id)
    sender_extra = ExtraInfo.objects.get(user=sender_user)
    
    # Find receiver
    receiver_designation = Designation.objects.get(name=receiver_designation_name)
    
    if receiver_department_id:
        receiver_dept = DepartmentInfo.objects.get(id=receiver_department_id)
        # Find user with this designation in this department
        potential_receivers = HoldsDesignation.objects.filter(
            designation=receiver_designation
        ).select_related('working__extrainfo')
        
        receiver_hd = None
        for hd in potential_receivers:
            try:
                extra = ExtraInfo.objects.get(user=hd.working)
                if extra.department == receiver_dept:
                    receiver_hd = hd
                    break
            except ExtraInfo.DoesNotExist:
                continue
    else:
        receiver_hd = HoldsDesignation.objects.filter(
            designation=receiver_designation
        ).first()
        receiver_dept = ExtraInfo.objects.get(user=receiver_hd.working).department if receiver_hd else file.current_department
    
    if not receiver_hd:
        raise ValueError(f"No user found with designation: {receiver_designation_name}")
    
    receiver_extra = ExtraInfo.objects.get(user=receiver_hd.working)
    
    # Get sender's current designation
    sender_hd = HoldsDesignation.objects.filter(working=sender_user).first()
    
    # Create movement record
    FileMovement.objects.create(
        file=file,
        action='FORWARD',
        sender=sender_extra,
        sender_designation=sender_hd.designation if sender_hd else file.current_designation,
        sender_department=sender_extra.department,
        receiver=receiver_extra,
        receiver_designation=receiver_designation,
        receiver_department=receiver_dept,
        remarks=remarks
    )
    
    # Update file current holder
    file.current_holder = receiver_extra
    file.current_designation = receiver_designation
    file.current_department = receiver_dept
    file.status = 'FORWARDED'
    file.save()
    
    return file

def get_file_history(file_id):
    """Get complete movement history of a file"""
    file = File.objects.get(id=file_id)
    
    movements = FileMovement.objects.filter(file=file).select_related(
        'sender', 'receiver', 
        'sender_designation', 'receiver_designation',
        'sender_department', 'receiver_department'
    ).order_by('timestamp')
    
    return {
        'file': file,
        'movements': list(movements),
        'total_movements': movements.count(),
        'days_open': (timezone.now() - file.created_at).days if file.status != 'CLOSED' else None
    }
```

### Generate File Number
```python
def generate_file_number(prefix, department):
    """Generate unique file number"""
    import datetime
    
    year = datetime.datetime.now().year
    
    # Get department code
    dept_code = department.name[:3].upper() if department else 'GEN'
    
    # Get next sequence
    last_file = File.objects.filter(
        file_number__startswith=f"{prefix}/{dept_code}/{year}"
    ).order_by('-file_number').first()
    
    if last_file:
        try:
            last_seq = int(last_file.file_number.split('/')[-1])
            next_seq = last_seq + 1
        except:
            next_seq = 1
    else:
        next_seq = 1
    
    return f"{prefix}/{dept_code}/{year}/{next_seq:04d}"
```

---

## API Endpoints

### New Endpoints to Create
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/fts/api/files/` | GET, POST | List/Create files |
| `/fts/api/files/<id>/` | GET, PUT | File details |
| `/fts/api/files/<id>/forward/` | POST | Forward file |
| `/fts/api/files/<id>/approve/` | POST | Approve file |
| `/fts/api/files/<id>/reject/` | POST | Reject file |
| `/fts/api/files/<id>/close/` | POST | Close file |
| `/fts/api/files/<id>/history/` | GET | File history |
| `/fts/api/inbox/` | GET | User's inbox |
| `/fts/api/outbox/` | GET | User's sent files |
| `/fts/api/pending/` | GET | Pending actions |
| `/fts/api/drafts/` | GET, POST | Draft files |
| `/fts/api/file-types/` | GET | Available file types |
| `/fts/api/designations/` | GET | Routing designations |

---

## Views Integration

```python
from django.contrib.auth.decorators import login_required
from applications.globals.models import ExtraInfo, HoldsDesignation

@login_required
def fts_inbox(request):
    """Show files in user's inbox"""
    user = request.user
    extrainfo = ExtraInfo.objects.get(user=user)
    
    # Get user's designations
    designations = HoldsDesignation.objects.filter(working=user)
    
    # Get files where user is current holder
    inbox_files = File.objects.filter(
        current_holder=extrainfo,
        status__in=['CREATED', 'IN_PROGRESS', 'FORWARDED', 'PENDING']
    ).select_related('file_type', 'created_by', 'source_department')
    
    return render(request, 'fts/inbox.html', {
        'files': inbox_files,
        'designations': designations,
    })

@login_required
def fts_outbox(request):
    """Show files sent by user"""
    user = request.user
    extrainfo = ExtraInfo.objects.get(user=user)
    
    # Get files created by user
    created_files = File.objects.filter(
        created_by=extrainfo
    ).exclude(status='ARCHIVED')
    
    # Get files forwarded by user
    sent_movements = FileMovement.objects.filter(
        sender=extrainfo,
        action='FORWARD'
    ).values_list('file_id', flat=True).distinct()
    
    sent_files = File.objects.filter(id__in=sent_movements)
    
    return render(request, 'fts/outbox.html', {
        'created_files': created_files,
        'sent_files': sent_files,
    })
```

---

## Important Sync Points

1. **Designation Mapping**: FTS routing depends on `HoldsDesignation` table - ensure designations are properly assigned.

2. **Department Routing**: Use `DepartmentInfo` for inter-department file movement.

3. **Module Integration**: Create files automatically when processes start in other modules (branch change, thesis, etc.).

4. **Status Sync**: When FTS file status changes, update corresponding source module records.

5. **Notification**: Send notifications on file movement using notification module.

---

