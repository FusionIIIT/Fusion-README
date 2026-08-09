# IWD (Institute Works Department) Module - Integration Guide

**What this module is:** The IWD (Institute Works Department) module will manage infrastructure - construction projects, maintenance requests, electrical/civil works, contractor management, work orders, and building/asset management.

**Why we need to build this:** Institute infrastructure needs constant maintenance and development but currently work requests are manual, tracking is on paper, contractor management is fragmented. This module will provide digital work order management, project tracking, and maintenance request workflows.

**Why integration is needed:** Work requests come from departments (DepartmentInfo) and employees (ExtraInfo/Faculty/Staff). Approvals flow through organizational hierarchy. The module needs organizational structure from globals to route requests correctly.

**Key dependencies:** ExtraInfo, Faculty, Staff, Discipline, DepartmentInfo

---

## Module Overview
The IWD module manages all civil works, electrical works, construction projects, maintenance, contractor management, and infrastructure development activities of the institute.

---

## Core Dependencies - Tables to Sync With

### 1. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in IWD |
|--------|------|-------------|--------------|
| `id` | CharField(20) | User ID | **Requester ID** |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | User type |
| `department` | ForeignKey(DepartmentInfo) | Department | **Requesting department** |

#### Table: `Faculty`
| Column | Type | Description | Usage in IWD |
|--------|------|-------------|--------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Work requestor** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in IWD |
|--------|------|-------------|--------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | **Work location department** |

#### Table: `Designation`
| Column | Type | Description | Usage in IWD |
|--------|------|-------------|--------------|
| `id` | AutoField | Primary Key | Designation reference |
| `name` | CharField(50) | Designation | **Approval authority** |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in IWD |
|--------|------|-------------|--------------|
| `user` | ForeignKey(User) | User | Designation holder |
| `designation` | ForeignKey(Designation) | Designation | **JE/AE/EE approval** |

---

## Data Models to Create in IWD Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, DepartmentInfo, Designation, HoldsDesignation

# ==================== INFRASTRUCTURE ====================

class Building(models.Model):
    """Institute buildings"""
    BUILDING_TYPE = [
        ('ACADEMIC', 'Academic'),
        ('ADMINISTRATIVE', 'Administrative'),
        ('HOSTEL', 'Hostel'),
        ('RESIDENTIAL', 'Residential'),
        ('UTILITY', 'Utility'),
        ('SPORTS', 'Sports'),
        ('OTHER', 'Other'),
    ]
    
    name = models.CharField(max_length=200)
    code = models.CharField(max_length=20, unique=True)
    building_type = models.CharField(max_length=20, choices=BUILDING_TYPE)
    
    # Location
    location = models.CharField(max_length=200, blank=True)
    area_sqft = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    
    # Construction
    year_constructed = models.IntegerField(null=True, blank=True)
    floors = models.IntegerField(default=1)
    
    # Department
    department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    is_active = models.BooleanField(default=True)

class Room(models.Model):
    """Rooms within buildings"""
    ROOM_TYPE = [
        ('CLASSROOM', 'Classroom'),
        ('LAB', 'Laboratory'),
        ('OFFICE', 'Office'),
        ('CONFERENCE', 'Conference Room'),
        ('STORE', 'Store Room'),
        ('TOILET', 'Toilet'),
        ('UTILITY', 'Utility'),
        ('OTHER', 'Other'),
    ]
    
    building = models.ForeignKey(Building, on_delete=models.CASCADE, related_name='rooms')
    room_number = models.CharField(max_length=20)
    room_type = models.CharField(max_length=20, choices=ROOM_TYPE)
    
    floor = models.IntegerField(default=0)
    area_sqft = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    class Meta:
        unique_together = ['building', 'room_number']

# ==================== CONTRACTOR MANAGEMENT ====================

class Contractor(models.Model):
    """Registered contractors"""
    CONTRACTOR_TYPE = [
        ('CIVIL', 'Civil'),
        ('ELECTRICAL', 'Electrical'),
        ('PLUMBING', 'Plumbing'),
        ('HVAC', 'HVAC'),
        ('GENERAL', 'General'),
    ]
    
    CATEGORY = [
        ('A', 'Category A'),
        ('B', 'Category B'),
        ('C', 'Category C'),
    ]
    
    name = models.CharField(max_length=200)
    contractor_type = models.CharField(max_length=20, choices=CONTRACTOR_TYPE)
    category = models.CharField(max_length=1, choices=CATEGORY)
    
    # Registration
    registration_number = models.CharField(max_length=50, unique=True)
    valid_until = models.DateField()
    
    # Contact
    contact_person = models.CharField(max_length=100)
    phone = models.CharField(max_length=15)
    email = models.EmailField()
    address = models.TextField()
    
    # Documents
    gst_number = models.CharField(max_length=20, blank=True)
    pan_number = models.CharField(max_length=15, blank=True)
    
    # Bank
    bank_name = models.CharField(max_length=100, blank=True)
    bank_account = models.CharField(max_length=30, blank=True)
    ifsc_code = models.CharField(max_length=15, blank=True)
    
    # Performance
    rating = models.DecimalField(max_digits=3, decimal_places=2, default=0)
    
    is_active = models.BooleanField(default=True)

# ==================== WORK REQUEST ====================

class WorkRequest(models.Model):
    """Work requests from departments"""
    STATUS = [
        ('SUBMITTED', 'Submitted'),
        ('UNDER_REVIEW', 'Under Review'),
        ('SITE_VISIT', 'Site Visit Scheduled'),
        ('ESTIMATE_PREPARED', 'Estimate Prepared'),
        ('APPROVED', 'Approved'),
        ('TENDERING', 'Under Tendering'),
        ('IN_PROGRESS', 'In Progress'),
        ('COMPLETED', 'Completed'),
        ('REJECTED', 'Rejected'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    WORK_TYPE = [
        ('CIVIL', 'Civil'),
        ('ELECTRICAL', 'Electrical'),
        ('PLUMBING', 'Plumbing'),
        ('CARPENTRY', 'Carpentry'),
        ('PAINTING', 'Painting'),
        ('HVAC', 'HVAC'),
        ('MAINTENANCE', 'Maintenance'),
        ('NEW_CONSTRUCTION', 'New Construction'),
        ('OTHER', 'Other'),
    ]
    
    PRIORITY = [
        ('LOW', 'Low'),
        ('MEDIUM', 'Medium'),
        ('HIGH', 'High'),
        ('URGENT', 'Urgent'),
    ]
    
    # Identification
    request_number = models.CharField(max_length=50, unique=True)
    
    # Requester
    requester = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='work_requests')
    requesting_department = models.ForeignKey(DepartmentInfo, on_delete=models.CASCADE)
    
    # Work details
    work_type = models.CharField(max_length=20, choices=WORK_TYPE)
    subject = models.CharField(max_length=300)
    description = models.TextField()
    
    # Location
    building = models.ForeignKey(Building, on_delete=models.SET_NULL, null=True, blank=True)
    room = models.ForeignKey(Room, on_delete=models.SET_NULL, null=True, blank=True)
    location_details = models.CharField(max_length=200, blank=True)
    
    # Priority
    priority = models.CharField(max_length=20, choices=PRIORITY, default='MEDIUM')
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='SUBMITTED')
    
    # Timeline
    submitted_at = models.DateTimeField(auto_now_add=True)
    required_by = models.DateField(null=True, blank=True)
    
    # Attachments
    sketch = models.FileField(upload_to='iwd/requests/sketches/', blank=True)
    
    # Processing
    assigned_to = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='assigned_work_requests')

class WorkRequestComment(models.Model):
    """Comments on work requests"""
    request = models.ForeignKey(WorkRequest, on_delete=models.CASCADE, related_name='comments')
    comment = models.TextField()
    commented_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    commented_at = models.DateTimeField(auto_now_add=True)
    attachment = models.FileField(upload_to='iwd/requests/comments/', blank=True)

# ==================== SITE VISIT ====================

class SiteVisit(models.Model):
    """Site visits for work requests"""
    STATUS = [
        ('SCHEDULED', 'Scheduled'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    work_request = models.ForeignKey(WorkRequest, on_delete=models.CASCADE, related_name='site_visits')
    
    scheduled_date = models.DateField()
    scheduled_time = models.TimeField(null=True, blank=True)
    
    visited_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='SCHEDULED')
    
    # Findings
    findings = models.TextField(blank=True)
    recommendations = models.TextField(blank=True)
    photos = models.FileField(upload_to='iwd/site_visits/', blank=True)
    
    visited_at = models.DateTimeField(null=True, blank=True)

# ==================== ESTIMATE ====================

class Estimate(models.Model):
    """Work estimates"""
    STATUS = [
        ('DRAFT', 'Draft'),
        ('SUBMITTED', 'Submitted'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('REVISED', 'Revised'),
    ]
    
    work_request = models.ForeignKey(WorkRequest, on_delete=models.CASCADE, related_name='estimates')
    
    estimate_number = models.CharField(max_length=50, unique=True)
    prepared_date = models.DateField()
    
    # Prepared by
    prepared_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='estimates_prepared')
    
    # Amounts
    civil_cost = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    electrical_cost = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    plumbing_cost = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    other_cost = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    contingency = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    total_cost = models.DecimalField(max_digits=15, decimal_places=2)
    
    # Documents
    detailed_estimate = models.FileField(upload_to='iwd/estimates/')
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='DRAFT')
    
    # Approval
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='estimates_approved')
    approved_at = models.DateTimeField(null=True, blank=True)
    approval_remarks = models.TextField(blank=True)

class EstimateItem(models.Model):
    """Line items in estimate"""
    estimate = models.ForeignKey(Estimate, on_delete=models.CASCADE, related_name='items')
    
    description = models.CharField(max_length=500)
    quantity = models.DecimalField(max_digits=12, decimal_places=2)
    unit = models.CharField(max_length=20)
    rate = models.DecimalField(max_digits=12, decimal_places=2)
    amount = models.DecimalField(max_digits=15, decimal_places=2)
    
    remarks = models.TextField(blank=True)

# ==================== TENDER ====================

class Tender(models.Model):
    """Tender for works"""
    STATUS = [
        ('DRAFT', 'Draft'),
        ('PUBLISHED', 'Published'),
        ('BID_OPENING', 'Bid Opening'),
        ('EVALUATION', 'Evaluation'),
        ('AWARDED', 'Awarded'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    TENDER_TYPE = [
        ('OPEN', 'Open Tender'),
        ('LIMITED', 'Limited Tender'),
        ('SINGLE', 'Single Tender'),
    ]
    
    work_request = models.ForeignKey(WorkRequest, on_delete=models.CASCADE, related_name='tenders')
    estimate = models.ForeignKey(Estimate, on_delete=models.CASCADE)
    
    tender_number = models.CharField(max_length=50, unique=True)
    tender_type = models.CharField(max_length=20, choices=TENDER_TYPE)
    
    title = models.CharField(max_length=300)
    description = models.TextField()
    
    # Timeline
    published_date = models.DateField(null=True, blank=True)
    last_date_submission = models.DateTimeField()
    bid_opening_date = models.DateTimeField()
    
    # Amounts
    estimated_cost = models.DecimalField(max_digits=15, decimal_places=2)
    emd_amount = models.DecimalField(max_digits=12, decimal_places=2)  # Earnest Money Deposit
    
    # Documents
    tender_document = models.FileField(upload_to='iwd/tenders/')
    
    status = models.CharField(max_length=20, choices=STATUS, default='DRAFT')
    
    created_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)

class TenderBid(models.Model):
    """Contractor bids"""
    STATUS = [
        ('SUBMITTED', 'Submitted'),
        ('OPENED', 'Opened'),
        ('EVALUATED', 'Evaluated'),
        ('SELECTED', 'Selected'),
        ('REJECTED', 'Rejected'),
    ]
    
    tender = models.ForeignKey(Tender, on_delete=models.CASCADE, related_name='bids')
    contractor = models.ForeignKey(Contractor, on_delete=models.CASCADE)
    
    bid_amount = models.DecimalField(max_digits=15, decimal_places=2)
    submission_date = models.DateTimeField()
    
    # Documents
    technical_bid = models.FileField(upload_to='iwd/bids/technical/')
    financial_bid = models.FileField(upload_to='iwd/bids/financial/')
    
    # EMD
    emd_paid = models.BooleanField(default=False)
    emd_reference = models.CharField(max_length=100, blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='SUBMITTED')
    
    # Evaluation
    technical_score = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)
    remarks = models.TextField(blank=True)

# ==================== WORK ORDER ====================

class WorkOrder(models.Model):
    """Work orders to contractors"""
    STATUS = [
        ('ISSUED', 'Issued'),
        ('ACKNOWLEDGED', 'Acknowledged'),
        ('IN_PROGRESS', 'In Progress'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    work_request = models.ForeignKey(WorkRequest, on_delete=models.CASCADE, related_name='work_orders')
    tender = models.ForeignKey(Tender, on_delete=models.SET_NULL, null=True, blank=True)
    contractor = models.ForeignKey(Contractor, on_delete=models.CASCADE)
    
    work_order_number = models.CharField(max_length=50, unique=True)
    work_order_date = models.DateField()
    
    # Work details
    scope_of_work = models.TextField()
    
    # Amounts
    contract_amount = models.DecimalField(max_digits=15, decimal_places=2)
    
    # Timeline
    start_date = models.DateField()
    completion_date = models.DateField()
    actual_completion_date = models.DateField(null=True, blank=True)
    
    # Documents
    work_order_document = models.FileField(upload_to='iwd/work_orders/')
    
    status = models.CharField(max_length=20, choices=STATUS, default='ISSUED')
    
    issued_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)

class WorkProgress(models.Model):
    """Work progress reports"""
    work_order = models.ForeignKey(WorkOrder, on_delete=models.CASCADE, related_name='progress_reports')
    
    report_date = models.DateField()
    progress_percentage = models.IntegerField()
    
    work_done_description = models.TextField()
    materials_used = models.TextField(blank=True)
    
    # Issues
    issues_faced = models.TextField(blank=True)
    
    # Site photos
    photos = models.FileField(upload_to='iwd/progress/', blank=True)
    
    reported_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)

# ==================== BILLING ====================

class RunningAccountBill(models.Model):
    """Running Account (RA) Bills"""
    STATUS = [
        ('SUBMITTED', 'Submitted'),
        ('VERIFIED', 'Verified'),
        ('APPROVED', 'Approved'),
        ('PAID', 'Paid'),
        ('REJECTED', 'Rejected'),
    ]
    
    work_order = models.ForeignKey(WorkOrder, on_delete=models.CASCADE, related_name='ra_bills')
    
    bill_number = models.CharField(max_length=50, unique=True)
    bill_date = models.DateField()
    
    # Amounts
    gross_amount = models.DecimalField(max_digits=15, decimal_places=2)
    previous_payments = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    deductions = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    net_amount = models.DecimalField(max_digits=15, decimal_places=2)
    
    # Measurement book reference
    mb_reference = models.CharField(max_length=100, blank=True)
    
    # Documents
    bill_document = models.FileField(upload_to='iwd/bills/')
    
    status = models.CharField(max_length=20, choices=STATUS, default='SUBMITTED')
    
    # Verification
    verified_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='bills_verified')
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='bills_approved')
    
    payment_date = models.DateField(null=True, blank=True)
    payment_reference = models.CharField(max_length=100, blank=True)

# ==================== MAINTENANCE ====================

class MaintenanceSchedule(models.Model):
    """Scheduled maintenance"""
    FREQUENCY = [
        ('DAILY', 'Daily'),
        ('WEEKLY', 'Weekly'),
        ('MONTHLY', 'Monthly'),
        ('QUARTERLY', 'Quarterly'),
        ('YEARLY', 'Yearly'),
    ]
    
    building = models.ForeignKey(Building, on_delete=models.CASCADE, related_name='maintenance_schedules')
    
    task_name = models.CharField(max_length=200)
    description = models.TextField()
    frequency = models.CharField(max_length=20, choices=FREQUENCY)
    
    assigned_to = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    next_due_date = models.DateField()
    last_completed = models.DateField(null=True, blank=True)
    
    is_active = models.BooleanField(default=True)

class MaintenanceLog(models.Model):
    """Maintenance activity log"""
    schedule = models.ForeignKey(MaintenanceSchedule, on_delete=models.CASCADE, related_name='logs', null=True, blank=True)
    building = models.ForeignKey(Building, on_delete=models.CASCADE)
    
    task_description = models.TextField()
    performed_date = models.DateField()
    performed_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    
    remarks = models.TextField(blank=True)
    cost = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
```

---

## Integration Functions

### Generate Request Number
```python
def generate_request_number(work_type):
    """Generate unique work request number"""
    from datetime import date
    
    today = date.today()
    prefix = f"IWD/{work_type[:3].upper()}/{today.strftime('%Y%m')}"
    
    last_request = WorkRequest.objects.filter(
        request_number__startswith=prefix
    ).order_by('-request_number').first()
    
    if last_request:
        last_seq = int(last_request.request_number.split('/')[-1])
        new_seq = last_seq + 1
    else:
        new_seq = 1
    
    return f"{prefix}/{new_seq:04d}"
```

### Get Approval Authority
```python
def get_approval_authority(estimate_amount):
    """Get approval authority based on estimate amount"""
    from applications.globals.models import HoldsDesignation, Designation
    
    # Define approval limits
    if estimate_amount <= 100000:
        designation_name = 'Junior Engineer'
    elif estimate_amount <= 500000:
        designation_name = 'Assistant Engineer'
    elif estimate_amount <= 2500000:
        designation_name = 'Executive Engineer'
    else:
        designation_name = 'Superintending Engineer'
    
    designation = Designation.objects.filter(name__icontains=designation_name).first()
    if designation:
        holder = HoldsDesignation.objects.filter(designation=designation).first()
        return holder.working if holder else None
    
    return None
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/iwd/api/requests/` | GET, POST | Work requests |
| `/iwd/api/requests/<id>/` | GET, PUT | Request details |
| `/iwd/api/estimates/` | GET, POST | Estimates |
| `/iwd/api/tenders/` | GET | Tenders |
| `/iwd/api/work-orders/` | GET | Work orders |
| `/iwd/api/bills/` | GET | RA Bills |
| `/iwd/api/buildings/` | GET | Buildings |

---

## Important Sync Points

1. **Requester**: Use `ExtraInfo` for requester identification.
2. **Department**: Get from `ExtraInfo.department` for routing.
3. **Approval**: Use `HoldsDesignation` to find JE/AE/EE for approvals.
4. **Budget**: Link to Finance module for budget verification.

---

