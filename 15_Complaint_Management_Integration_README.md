# Complaint Management Module - Integration Guide

**What this module is:** The Complaint Management module will handle grievance redressal - complaint registration, categorization (academic/hostel/mess/infrastructure), assignment to handlers, tracking, resolution, and feedback collection.

**Why we need to build this:** Students and staff face issues daily - hostel problems, academic grievances, infrastructure complaints. Currently complaints are verbal or via scattered emails with no tracking. This module will provide structured complaint registration, automatic routing, and resolution tracking.

**Why integration is needed:** Complainants are students (Student) or employees (ExtraInfo/Staff). Complaints are routed to departments (DepartmentInfo) or specific authorities. The module needs identity data to know who is complaining and who should resolve it.

**Key dependencies:** Student, ExtraInfo, Faculty, Staff, Batch, Discipline

---

## Module Overview
The Complaint Management module handles grievance registration, tracking, resolution, and feedback for various complaint categories across the institute - including academic, hostel, mess, infrastructure, and administrative complaints.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Batch`
| Column | Type | Description | Usage in Complaint |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Complainant batch |
| `name` | CharField(50) | Batch name | Batch info |

#### Table: `Discipline`
| Column | Type | Description | Usage in Complaint |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Branch reference |
| `name` | CharField(100) | Discipline name | Department routing |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in Complaint |
|--------|------|-------------|-------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Student complainant** |
| `batch_id` | ForeignKey(Batch) | Batch | Batch context |

---

### 3. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Complaint |
|--------|------|-------------|-------------------|
| `id` | CharField(20) | User ID | **Complainant ID** |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | **Complainant type** |
| `phone_no` | BigIntegerField | Phone | **Contact** |
| `department` | ForeignKey(DepartmentInfo) | Department | **Routing** |

#### Table: `Faculty`
| Column | Type | Description | Usage in Complaint |
|--------|------|-------------|-------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Faculty complainant** |

#### Table: `Staff`
| Column | Type | Description | Usage in Complaint |
|--------|------|-------------|-------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Staff complainant** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in Complaint |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | **Complaint routing** |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in Complaint |
|--------|------|-------------|-------------------|
| `user` | ForeignKey(User) | User | Designation holder |
| `designation` | ForeignKey(Designation) | Designation | **Resolver authority** |

---

## Data Models to Create in Complaint Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, Faculty, Staff, DepartmentInfo, Designation, HoldsDesignation
from applications.academic_information.models import Student

# ==================== COMPLAINT CONFIGURATION ====================

class ComplaintCategory(models.Model):
    """Categories of complaints"""
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=20, unique=True)
    description = models.TextField(blank=True)
    
    # Routing
    default_handler_designation = models.ForeignKey(Designation, on_delete=models.SET_NULL, null=True, blank=True)
    default_department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    # SLA
    resolution_days = models.IntegerField(default=7)  # Expected resolution time
    escalation_days = models.IntegerField(default=3)  # Escalate after these days
    
    is_active = models.BooleanField(default=True)
    
    class Meta:
        verbose_name_plural = "Complaint Categories"

class ComplaintSubCategory(models.Model):
    """Sub-categories of complaints"""
    category = models.ForeignKey(ComplaintCategory, on_delete=models.CASCADE, related_name='subcategories')
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=30)
    description = models.TextField(blank=True)
    
    # Specific routing
    handler_designation = models.ForeignKey(Designation, on_delete=models.SET_NULL, null=True, blank=True)
    
    is_active = models.BooleanField(default=True)
    
    class Meta:
        unique_together = ['category', 'code']

# ==================== COMPLAINT MANAGEMENT ====================

class Complaint(models.Model):
    """Main complaint records"""
    STATUS = [
        ('SUBMITTED', 'Submitted'),
        ('ACKNOWLEDGED', 'Acknowledged'),
        ('IN_PROGRESS', 'In Progress'),
        ('ON_HOLD', 'On Hold'),
        ('RESOLVED', 'Resolved'),
        ('CLOSED', 'Closed'),
        ('REOPENED', 'Reopened'),
        ('REJECTED', 'Rejected'),
    ]
    
    PRIORITY = [
        ('LOW', 'Low'),
        ('MEDIUM', 'Medium'),
        ('HIGH', 'High'),
        ('CRITICAL', 'Critical'),
    ]
    
    # Identification
    complaint_number = models.CharField(max_length=50, unique=True)
    
    # Complainant (from globals)
    complainant = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='complaints')
    complainant_department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    # Category
    category = models.ForeignKey(ComplaintCategory, on_delete=models.PROTECT)
    subcategory = models.ForeignKey(ComplaintSubCategory, on_delete=models.SET_NULL, null=True, blank=True)
    
    # Details
    subject = models.CharField(max_length=300)
    description = models.TextField()
    
    # Location (if applicable)
    location = models.CharField(max_length=200, blank=True)
    
    # Priority
    priority = models.CharField(max_length=20, choices=PRIORITY, default='MEDIUM')
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='SUBMITTED')
    
    # Assignment
    assigned_to = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='assigned_complaints')
    assigned_department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='department_complaints')
    
    # Timeline
    submitted_at = models.DateTimeField(auto_now_add=True)
    acknowledged_at = models.DateTimeField(null=True, blank=True)
    resolved_at = models.DateTimeField(null=True, blank=True)
    closed_at = models.DateTimeField(null=True, blank=True)
    
    # SLA
    expected_resolution = models.DateTimeField(null=True, blank=True)
    is_sla_breached = models.BooleanField(default=False)
    
    # Escalation
    escalation_level = models.IntegerField(default=0)
    
    # Anonymous option
    is_anonymous = models.BooleanField(default=False)
    
    class Meta:
        ordering = ['-submitted_at']

class ComplaintAttachment(models.Model):
    """Attachments for complaints"""
    complaint = models.ForeignKey(Complaint, on_delete=models.CASCADE, related_name='attachments')
    file = models.FileField(upload_to='complaints/attachments/')
    file_name = models.CharField(max_length=200)
    uploaded_at = models.DateTimeField(auto_now_add=True)
    uploaded_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)

# ==================== WORKFLOW ====================

class ComplaintStatusHistory(models.Model):
    """Track status changes"""
    complaint = models.ForeignKey(Complaint, on_delete=models.CASCADE, related_name='status_history')
    
    from_status = models.CharField(max_length=20)
    to_status = models.CharField(max_length=20)
    
    changed_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    changed_at = models.DateTimeField(auto_now_add=True)
    
    remarks = models.TextField(blank=True)

class ComplaintComment(models.Model):
    """Comments/Notes on complaints"""
    COMMENT_TYPE = [
        ('PUBLIC', 'Public'),  # Visible to complainant
        ('INTERNAL', 'Internal'),  # Staff only
    ]
    
    complaint = models.ForeignKey(Complaint, on_delete=models.CASCADE, related_name='comments')
    
    comment_type = models.CharField(max_length=20, choices=COMMENT_TYPE, default='PUBLIC')
    content = models.TextField()
    
    commented_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    commented_at = models.DateTimeField(auto_now_add=True)
    
    # Attachment
    attachment = models.FileField(upload_to='complaints/comments/', blank=True)

class ComplaintAssignmentHistory(models.Model):
    """Track assignment changes"""
    complaint = models.ForeignKey(Complaint, on_delete=models.CASCADE, related_name='assignment_history')
    
    from_assignee = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='reassigned_from')
    to_assignee = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='reassigned_to')
    
    assigned_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='assignments_made')
    assigned_at = models.DateTimeField(auto_now_add=True)
    
    reason = models.TextField(blank=True)

# ==================== RESOLUTION ====================

class ComplaintResolution(models.Model):
    """Complaint resolution details"""
    complaint = models.OneToOneField(Complaint, on_delete=models.CASCADE, related_name='resolution')
    
    resolution_summary = models.TextField()
    action_taken = models.TextField()
    
    resolved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    resolved_at = models.DateTimeField(auto_now_add=True)
    
    # Root cause
    root_cause = models.TextField(blank=True)
    preventive_measures = models.TextField(blank=True)
    
    # Supporting documents
    resolution_document = models.FileField(upload_to='complaints/resolutions/', blank=True)

# ==================== FEEDBACK ====================

class ComplaintFeedback(models.Model):
    """Complainant feedback after resolution"""
    complaint = models.OneToOneField(Complaint, on_delete=models.CASCADE, related_name='feedback')
    
    # Ratings (1-5)
    response_time_rating = models.IntegerField()
    resolution_quality_rating = models.IntegerField()
    staff_behavior_rating = models.IntegerField()
    overall_rating = models.IntegerField()
    
    is_satisfied = models.BooleanField()
    comments = models.TextField(blank=True)
    
    submitted_at = models.DateTimeField(auto_now_add=True)

# ==================== ESCALATION ====================

class EscalationMatrix(models.Model):
    """Escalation rules"""
    category = models.ForeignKey(ComplaintCategory, on_delete=models.CASCADE, related_name='escalation_matrix')
    
    level = models.IntegerField()  # 1, 2, 3, etc.
    escalate_after_days = models.IntegerField()
    escalate_to = models.ForeignKey(Designation, on_delete=models.CASCADE)
    
    notification_required = models.BooleanField(default=True)
    
    class Meta:
        unique_together = ['category', 'level']
        ordering = ['category', 'level']

class EscalationRecord(models.Model):
    """Track escalations"""
    complaint = models.ForeignKey(Complaint, on_delete=models.CASCADE, related_name='escalations')
    
    escalation_level = models.IntegerField()
    escalated_to = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    escalated_at = models.DateTimeField(auto_now_add=True)
    
    reason = models.CharField(max_length=200)  # SLA breach, Priority increase, etc.
    
    action_taken = models.TextField(blank=True)
    action_taken_at = models.DateTimeField(null=True, blank=True)

# ==================== REPORTS ====================

class ComplaintReport(models.Model):
    """Periodic complaint reports"""
    REPORT_TYPE = [
        ('DAILY', 'Daily'),
        ('WEEKLY', 'Weekly'),
        ('MONTHLY', 'Monthly'),
        ('QUARTERLY', 'Quarterly'),
    ]
    
    report_type = models.CharField(max_length=20, choices=REPORT_TYPE)
    
    period_start = models.DateField()
    period_end = models.DateField()
    
    # Statistics
    total_received = models.IntegerField(default=0)
    total_resolved = models.IntegerField(default=0)
    total_pending = models.IntegerField(default=0)
    total_sla_breached = models.IntegerField(default=0)
    
    average_resolution_time = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)  # hours
    
    # Category-wise breakdown
    category_breakdown = models.TextField(blank=True)  # JSON
    
    generated_at = models.DateTimeField(auto_now_add=True)
    generated_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)

# ==================== QUICK COMPLAINT TYPES ====================

class MaintenanceComplaint(models.Model):
    """Infrastructure/Maintenance complaints"""
    AREA_TYPE = [
        ('ACADEMIC', 'Academic Building'),
        ('HOSTEL', 'Hostel'),
        ('MESS', 'Mess'),
        ('LIBRARY', 'Library'),
        ('GROUND', 'Sports Ground'),
        ('ROAD', 'Roads'),
        ('COMMON', 'Common Area'),
        ('OTHER', 'Other'),
    ]
    
    ISSUE_TYPE = [
        ('ELECTRICAL', 'Electrical'),
        ('PLUMBING', 'Plumbing'),
        ('FURNITURE', 'Furniture'),
        ('CIVIL', 'Civil Work'),
        ('HOUSEKEEPING', 'Housekeeping'),
        ('OTHER', 'Other'),
    ]
    
    complaint = models.OneToOneField(Complaint, on_delete=models.CASCADE, related_name='maintenance_details')
    
    area_type = models.CharField(max_length=20, choices=AREA_TYPE)
    issue_type = models.CharField(max_length=20, choices=ISSUE_TYPE)
    
    building_name = models.CharField(max_length=100, blank=True)
    room_number = models.CharField(max_length=50, blank=True)
    floor = models.CharField(max_length=20, blank=True)

class AcademicComplaint(models.Model):
    """Academic-related complaints"""
    ISSUE_TYPE = [
        ('RESULT', 'Result/Grades'),
        ('ATTENDANCE', 'Attendance'),
        ('SYLLABUS', 'Syllabus'),
        ('TEACHING', 'Teaching Quality'),
        ('EXAM', 'Examination'),
        ('REGISTRATION', 'Registration'),
        ('OTHER', 'Other'),
    ]
    
    complaint = models.OneToOneField(Complaint, on_delete=models.CASCADE, related_name='academic_details')
    
    issue_type = models.CharField(max_length=20, choices=ISSUE_TYPE)
    course_code = models.CharField(max_length=20, blank=True)
    course_name = models.CharField(max_length=200, blank=True)
    semester = models.CharField(max_length=20, blank=True)
```

---

## Integration Functions

### Generate Complaint Number
```python
def generate_complaint_number(category):
    """Generate unique complaint number"""
    from datetime import date
    
    today = date.today()
    prefix = f"CMP/{category.code}/{today.strftime('%Y%m')}"
    
    last_complaint = Complaint.objects.filter(
        complaint_number__startswith=prefix
    ).order_by('-complaint_number').first()
    
    if last_complaint:
        last_seq = int(last_complaint.complaint_number.split('/')[-1])
        new_seq = last_seq + 1
    else:
        new_seq = 1
    
    return f"{prefix}/{new_seq:04d}"
```

### Get Complainant Info
```python
def get_complainant_info(user):
    """Get complainant details"""
    from applications.globals.models import ExtraInfo
    from applications.academic_information.models import Student
    
    extrainfo = ExtraInfo.objects.get(user=user)
    
    info = {
        'id': extrainfo.id,
        'name': f"{user.first_name} {user.last_name}",
        'user_type': extrainfo.user_type,
        'phone': extrainfo.phone_no,
        'email': user.email,
        'department': extrainfo.department,
    }
    
    # Add student-specific info
    if extrainfo.user_type == 'student':
        try:
            student = Student.objects.get(id=extrainfo)
            info['batch'] = student.batch_id
            info['roll_no'] = extrainfo.id
        except Student.DoesNotExist:
            pass
    
    return info
```

### Auto-assign Complaint
```python
def auto_assign_complaint(complaint):
    """Auto-assign complaint based on category routing"""
    from applications.globals.models import HoldsDesignation
    
    # Get handler designation from category
    handler_designation = complaint.subcategory.handler_designation if complaint.subcategory else complaint.category.default_handler_designation
    
    if handler_designation:
        # Find user holding this designation
        holder = HoldsDesignation.objects.filter(
            designation=handler_designation
        ).first()
        
        if holder:
            complaint.assigned_to = holder.working
            complaint.save()
            return holder.working
    
    return None
```

### Calculate SLA
```python
def calculate_sla(complaint):
    """Calculate expected resolution time based on category and priority"""
    from datetime import timedelta
    from django.utils import timezone
    
    base_days = complaint.category.resolution_days
    
    # Adjust based on priority
    priority_multiplier = {
        'LOW': 1.5,
        'MEDIUM': 1.0,
        'HIGH': 0.5,
        'CRITICAL': 0.25,
    }
    
    adjusted_days = base_days * priority_multiplier.get(complaint.priority, 1.0)
    
    expected_resolution = timezone.now() + timedelta(days=adjusted_days)
    return expected_resolution
```

### Check and Trigger Escalation
```python
def check_escalation(complaint):
    """Check if complaint needs escalation"""
    from django.utils import timezone
    
    if complaint.status in ['RESOLVED', 'CLOSED', 'REJECTED']:
        return False
    
    # Get escalation matrix for this category
    escalations = EscalationMatrix.objects.filter(
        category=complaint.category,
        level__gt=complaint.escalation_level
    ).order_by('level')
    
    for escalation in escalations:
        # Check if time exceeded
        hours_since_submission = (timezone.now() - complaint.submitted_at).total_seconds() / 3600
        days_since_submission = hours_since_submission / 24
        
        if days_since_submission >= escalation.escalate_after_days:
            # Trigger escalation
            from applications.globals.models import HoldsDesignation
            
            escalate_to = HoldsDesignation.objects.filter(
                designation=escalation.escalate_to
            ).first()
            
            if escalate_to:
                EscalationRecord.objects.create(
                    complaint=complaint,
                    escalation_level=escalation.level,
                    escalated_to=escalate_to.working,
                    reason='SLA breach'
                )
                
                complaint.escalation_level = escalation.level
                complaint.is_sla_breached = True
                complaint.save()
                
                return True
    
    return False
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/complaints/api/complaints/` | GET, POST | Complaint list/create |
| `/complaints/api/complaints/<id>/` | GET, PUT | Complaint details |
| `/complaints/api/complaints/<id>/assign/` | POST | Assign complaint |
| `/complaints/api/complaints/<id>/resolve/` | POST | Resolve complaint |
| `/complaints/api/complaints/<id>/feedback/` | POST | Submit feedback |
| `/complaints/api/categories/` | GET | Categories |
| `/complaints/api/dashboard/` | GET | Statistics dashboard |

---

## Important Sync Points

1. **Complainant**: Use `ExtraInfo` for all user types (student/faculty/staff).
2. **Department**: Get from `ExtraInfo.department` for routing.
3. **Assignment**: Use `HoldsDesignation` to find designated handlers.
4. **Student Info**: Get batch info from `Student.batch_id`.
5. **Anonymous Option**: Allow complaints without revealing identity.

---

