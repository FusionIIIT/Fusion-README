# Hostel Management Module - Integration Guide

**What this module is:** The Hostel Management module will handle student accommodation - room allocation, hostel transfers, leave requests, visitor entry, complaints, and hostel administration.

**Why we need to build this:** Residential students need hostel management. Currently room allocation is manual, leave requests are on paper. This module will provide automated room allocation based on batch/programme/gender, digital leave requests, and warden dashboards.

**Why integration is needed:** Hostel residents are students (from Student table). Room allocation considers batch and programme (from programme_curriculum). Student's current hostel info is stored in academic_information. The module will query core tables for student identity and academic information.

**Key dependencies:** Student, Batch, ExtraInfo, Dues (for hostel dues)

---

## Module Overview
The Hostel Management module handles student hostel allocation, room management, mess assignments, leave requests, complaints, and all hostel-related administrative functions.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Batch`
| Column | Type | Description | Usage in Hostel |
|--------|------|-------------|-----------------|
| `id` | AutoField | Primary Key | Student batch |
| `name` | CharField(50) | Batch name | **Year-wise allocation** |
| `discipline` | ForeignKey(Discipline) | Discipline | Programme context |
| `year` | IntegerField | Year | **Batch year** |

#### Table: `Discipline`
| Column | Type | Description | Usage in Hostel |
|--------|------|-------------|-----------------|
| `id` | AutoField | Primary Key | Programme reference |
| `name` | CharField(100) | Programme name | **Branch allocation rules** |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in Hostel |
|--------|------|-------------|-----------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Student hostel resident** |
| `batch_id` | ForeignKey(Batch) | Batch | **Batch-wise allocation** |

---

### 3. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Hostel |
|--------|------|-------------|-----------------|
| `id` | CharField(20) | User ID | **Resident ID (Roll No)** |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | Must be 'student' |
| `sex` | CharField(2) | Gender | **Boys/Girls hostel** |
| `phone_no` | BigIntegerField | Phone | Emergency contact |
| `address` | TextField | Address | Permanent address |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in Hostel |
|--------|------|-------------|-----------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | Warden department |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in Hostel |
|--------|------|-------------|-----------------|
| `user` | ForeignKey(User) | User | Designation holder |
| `designation` | ForeignKey(Designation) | Designation | **Warden/Caretaker** |

---

## Data Models to Create in Hostel Management Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, DepartmentInfo
from applications.academic_information.models import Student
from applications.programme_curriculum.models import Batch

# ==================== HOSTEL INFRASTRUCTURE ====================

class Hostel(models.Model):
    """Hostel buildings"""
    HOSTEL_TYPE = [
        ('BOYS', 'Boys Hostel'),
        ('GIRLS', 'Girls Hostel'),
        ('PHD', 'PhD Hostel'),
        ('MARRIED', 'Married Scholars'),
    ]
    
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=20, unique=True)
    hostel_type = models.CharField(max_length=20, choices=HOSTEL_TYPE)
    
    # Location
    location = models.CharField(max_length=200, blank=True)
    
    # Capacity
    total_rooms = models.IntegerField(default=0)
    total_beds = models.IntegerField(default=0)
    
    # Contact
    contact_number = models.CharField(max_length=15, blank=True)
    
    # Warden
    warden = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='warden_of')
    caretaker = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='caretaker_of')
    
    # Status
    is_active = models.BooleanField(default=True)
    
    class Meta:
        ordering = ['name']

class Wing(models.Model):
    """Hostel wings/blocks"""
    hostel = models.ForeignKey(Hostel, on_delete=models.CASCADE, related_name='wings')
    name = models.CharField(max_length=50)  # A, B, C or Wing-1, Wing-2
    floor_count = models.IntegerField(default=3)
    
    class Meta:
        unique_together = ['hostel', 'name']

class Room(models.Model):
    """Hostel rooms"""
    ROOM_TYPE = [
        ('SINGLE', 'Single'),
        ('DOUBLE', 'Double'),
        ('TRIPLE', 'Triple'),
        ('QUAD', 'Quad'),
    ]
    
    STATUS = [
        ('AVAILABLE', 'Available'),
        ('OCCUPIED', 'Occupied'),
        ('PARTIALLY_OCCUPIED', 'Partially Occupied'),
        ('MAINTENANCE', 'Under Maintenance'),
        ('RESERVED', 'Reserved'),
    ]
    
    hostel = models.ForeignKey(Hostel, on_delete=models.CASCADE, related_name='rooms')
    wing = models.ForeignKey(Wing, on_delete=models.CASCADE, null=True, blank=True)
    room_number = models.CharField(max_length=20)
    floor = models.IntegerField(default=0)
    
    room_type = models.CharField(max_length=20, choices=ROOM_TYPE, default='DOUBLE')
    capacity = models.IntegerField(default=2)
    current_occupancy = models.IntegerField(default=0)
    
    status = models.CharField(max_length=20, choices=STATUS, default='AVAILABLE')
    
    # Amenities
    has_attached_bathroom = models.BooleanField(default=False)
    has_ac = models.BooleanField(default=False)
    has_balcony = models.BooleanField(default=False)
    
    remarks = models.TextField(blank=True)
    
    class Meta:
        unique_together = ['hostel', 'room_number']
        ordering = ['hostel', 'room_number']

# ==================== STUDENT ALLOCATION ====================

class HostelAllotment(models.Model):
    """Student hostel allocation"""
    STATUS = [
        ('ACTIVE', 'Active'),
        ('VACATED', 'Vacated'),
        ('SUSPENDED', 'Suspended'),
    ]
    
    # Student reference (from academic_information)
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='hostel_allotments')
    
    # Room allocation
    room = models.ForeignKey(Room, on_delete=models.PROTECT, related_name='allotments')
    bed_number = models.CharField(max_length=10, blank=True)  # A, B, C or 1, 2, 3
    
    # Duration
    allotment_date = models.DateField()
    valid_until = models.DateField(null=True, blank=True)  # Academic year end
    
    # Batch info (from programme_curriculum)
    batch = models.ForeignKey(Batch, on_delete=models.SET_NULL, null=True)
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='ACTIVE')
    
    # Fees
    hostel_fee_paid = models.BooleanField(default=False)
    
    # Vacating
    vacated_date = models.DateField(null=True, blank=True)
    vacating_reason = models.TextField(blank=True)
    
    class Meta:
        ordering = ['-allotment_date']

class AllotmentHistory(models.Model):
    """Historical record of room changes"""
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='allotment_history')
    room = models.ForeignKey(Room, on_delete=models.SET_NULL, null=True)
    
    from_date = models.DateField()
    to_date = models.DateField()
    
    reason = models.TextField(blank=True)  # Transfer, Upgrade, Disciplinary, etc.

# ==================== ROOM CHANGE REQUESTS ====================

class RoomChangeRequest(models.Model):
    """Room change/transfer requests"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('COMPLETED', 'Completed'),
    ]
    
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='room_change_requests')
    current_room = models.ForeignKey(Room, on_delete=models.PROTECT, related_name='change_from')
    requested_room = models.ForeignKey(Room, on_delete=models.SET_NULL, null=True, blank=True, related_name='change_to')
    
    reason = models.TextField()
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    # Approval
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    approved_at = models.DateTimeField(null=True, blank=True)
    approval_remarks = models.TextField(blank=True)
    
    requested_at = models.DateTimeField(auto_now_add=True)

# ==================== LEAVE MANAGEMENT ====================

class HostelLeave(models.Model):
    """Hostel leave applications"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    LEAVE_TYPE = [
        ('WEEKEND', 'Weekend Leave'),
        ('VACATION', 'Vacation'),
        ('EMERGENCY', 'Emergency'),
        ('MEDICAL', 'Medical'),
        ('OTHER', 'Other'),
    ]
    
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='hostel_leaves')
    
    leave_type = models.CharField(max_length=20, choices=LEAVE_TYPE)
    from_date = models.DateField()
    to_date = models.DateField()
    
    destination = models.CharField(max_length=200)
    purpose = models.TextField()
    
    # Parent/Guardian details
    guardian_contact = models.CharField(max_length=15)
    guardian_consent = models.BooleanField(default=False)
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    # Approval
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    approved_at = models.DateTimeField(null=True, blank=True)
    remarks = models.TextField(blank=True)
    
    # Check-out/Check-in
    actual_departure = models.DateTimeField(null=True, blank=True)
    actual_return = models.DateTimeField(null=True, blank=True)
    
    applied_at = models.DateTimeField(auto_now_add=True)

# ==================== MESS ASSIGNMENT ====================

class MessInfo(models.Model):
    """Mess information"""
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=20, unique=True)
    location = models.CharField(max_length=200, blank=True)
    
    # Capacity
    capacity = models.IntegerField(default=500)
    
    # Type
    mess_type = models.CharField(max_length=50, blank=True)  # Veg, Non-Veg, Both
    
    # Contact
    contact_number = models.CharField(max_length=15, blank=True)
    mess_manager = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    is_active = models.BooleanField(default=True)

class MessAllotment(models.Model):
    """Student mess allocation"""
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='mess_allotments')
    mess = models.ForeignKey(MessInfo, on_delete=models.PROTECT)
    
    from_date = models.DateField()
    to_date = models.DateField(null=True, blank=True)
    
    is_active = models.BooleanField(default=True)
    
    class Meta:
        ordering = ['-from_date']

class MessRebate(models.Model):
    """Mess rebate applications"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
    ]
    
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='mess_rebates')
    
    from_date = models.DateField()
    to_date = models.DateField()
    
    reason = models.TextField()
    
    # Rebate amount
    days = models.IntegerField(default=0)
    rebate_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    approved_at = models.DateTimeField(null=True, blank=True)
    
    applied_at = models.DateTimeField(auto_now_add=True)

# ==================== COMPLAINTS ====================

class HostelComplaint(models.Model):
    """Hostel complaints"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('IN_PROGRESS', 'In Progress'),
        ('RESOLVED', 'Resolved'),
        ('CLOSED', 'Closed'),
    ]
    
    CATEGORY = [
        ('MAINTENANCE', 'Maintenance'),
        ('CLEANLINESS', 'Cleanliness'),
        ('ELECTRICAL', 'Electrical'),
        ('PLUMBING', 'Plumbing'),
        ('FURNITURE', 'Furniture'),
        ('INTERNET', 'Internet/WiFi'),
        ('SECURITY', 'Security'),
        ('MESS', 'Mess Related'),
        ('OTHER', 'Other'),
    ]
    
    PRIORITY = [
        ('LOW', 'Low'),
        ('MEDIUM', 'Medium'),
        ('HIGH', 'High'),
        ('URGENT', 'Urgent'),
    ]
    
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='hostel_complaints')
    room = models.ForeignKey(Room, on_delete=models.SET_NULL, null=True, blank=True)
    
    category = models.CharField(max_length=20, choices=CATEGORY)
    priority = models.CharField(max_length=20, choices=PRIORITY, default='MEDIUM')
    
    subject = models.CharField(max_length=200)
    description = models.TextField()
    
    # Attachments
    image = models.ImageField(upload_to='hostel/complaints/', blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    # Resolution
    assigned_to = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='assigned_complaints')
    resolution_remarks = models.TextField(blank=True)
    resolved_at = models.DateTimeField(null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

# ==================== NOTICES ====================

class HostelNotice(models.Model):
    """Hostel notices/announcements"""
    hostel = models.ForeignKey(Hostel, on_delete=models.CASCADE, null=True, blank=True)  # Null = all hostels
    
    title = models.CharField(max_length=200)
    content = models.TextField()
    
    # Target
    target_batch = models.ForeignKey(Batch, on_delete=models.SET_NULL, null=True, blank=True)
    
    # Validity
    posted_at = models.DateTimeField(auto_now_add=True)
    valid_until = models.DateField(null=True, blank=True)
    
    posted_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    is_active = models.BooleanField(default=True)

# ==================== ATTENDANCE ====================

class HostelAttendance(models.Model):
    """Student attendance in hostel"""
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='hostel_attendance')
    date = models.DateField()
    
    is_present = models.BooleanField(default=True)
    marked_at = models.TimeField(null=True, blank=True)
    marked_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    remarks = models.TextField(blank=True)
    
    class Meta:
        unique_together = ['student', 'date']

# ==================== FEES ====================

class HostelFee(models.Model):
    """Hostel fee structure"""
    hostel = models.ForeignKey(Hostel, on_delete=models.CASCADE)
    room_type = models.CharField(max_length=20)
    academic_year = models.CharField(max_length=10)  # e.g., "2024-25"
    
    room_rent = models.DecimalField(max_digits=10, decimal_places=2)
    mess_advance = models.DecimalField(max_digits=10, decimal_places=2)
    establishment_charges = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    electricity_charges = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    other_charges = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    total_fee = models.DecimalField(max_digits=10, decimal_places=2)
    
    class Meta:
        unique_together = ['hostel', 'room_type', 'academic_year']

class HostelFeePayment(models.Model):
    """Hostel fee payment records"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('PAID', 'Paid'),
        ('PARTIAL', 'Partially Paid'),
    ]
    
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='hostel_fee_payments')
    academic_year = models.CharField(max_length=10)
    semester = models.CharField(max_length=20)
    
    fee_structure = models.ForeignKey(HostelFee, on_delete=models.SET_NULL, null=True)
    amount_due = models.DecimalField(max_digits=10, decimal_places=2)
    amount_paid = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    payment_date = models.DateField(null=True, blank=True)
    transaction_id = models.CharField(max_length=100, blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    class Meta:
        unique_together = ['student', 'academic_year', 'semester']
```

---

## Integration Functions

### Get Student for Hostel Allocation
```python
def get_student_for_hostel(user):
    """Get student details for hostel allocation"""
    from applications.globals.models import ExtraInfo
    from applications.academic_information.models import Student
    from applications.programme_curriculum.models import Batch
    
    extrainfo = ExtraInfo.objects.get(user=user)
    
    if extrainfo.user_type != 'student':
        raise ValueError("User is not a student")
    
    student = Student.objects.select_related('batch_id').get(id=extrainfo)
    
    return {
        'student': student,
        'roll_no': extrainfo.id,
        'name': f"{user.first_name} {user.last_name}",
        'gender': extrainfo.sex,
        'batch': student.batch_id,
        'phone': extrainfo.phone_no,
    }
```

### Determine Hostel by Gender
```python
def get_eligible_hostels(gender):
    """Get hostels based on student gender"""
    if gender == 'M':
        return Hostel.objects.filter(hostel_type='BOYS', is_active=True)
    elif gender == 'F':
        return Hostel.objects.filter(hostel_type='GIRLS', is_active=True)
    return Hostel.objects.none()
```

### Check Room Availability
```python
def get_available_rooms(hostel, room_type=None):
    """Get available rooms in a hostel"""
    rooms = Room.objects.filter(
        hostel=hostel,
        status__in=['AVAILABLE', 'PARTIALLY_OCCUPIED']
    ).filter(current_occupancy__lt=models.F('capacity'))
    
    if room_type:
        rooms = rooms.filter(room_type=room_type)
    
    return rooms
```

### Allocate Room to Student
```python
def allocate_room(student, room, academic_year):
    """Allocate a room to student"""
    from datetime import date
    
    # Create allocation
    allotment = HostelAllotment.objects.create(
        student=student,
        room=room,
        batch=student.batch_id,
        allotment_date=date.today(),
        status='ACTIVE'
    )
    
    # Update room occupancy
    room.current_occupancy += 1
    if room.current_occupancy >= room.capacity:
        room.status = 'OCCUPIED'
    else:
        room.status = 'PARTIALLY_OCCUPIED'
    room.save()
    
    return allotment
```

### Get Students by Batch for Allocation
```python
def get_batch_students_for_allotment(batch_id):
    """Get students of a batch without hostel allocation"""
    from applications.academic_information.models import Student
    
    # Get students without active allotment
    students = Student.objects.filter(batch_id=batch_id).exclude(
        hostel_allotments__status='ACTIVE'
    ).select_related('id__user')
    
    return students
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/hostel/api/hostels/` | GET | Hostel list |
| `/hostel/api/rooms/` | GET | Room list |
| `/hostel/api/rooms/available/` | GET | Available rooms |
| `/hostel/api/allotments/` | GET, POST | Student allotments |
| `/hostel/api/leave/` | GET, POST | Leave applications |
| `/hostel/api/mess/` | GET | Mess info |
| `/hostel/api/complaints/` | GET, POST | Complaints |
| `/hostel/api/notices/` | GET | Notices |

---

## Important Sync Points

1. **Student Base**: Always use `Student` (linked to `ExtraInfo`) as resident reference.
2. **Gender**: Determine hostel type from `ExtraInfo.sex`.
3. **Batch**: Use `Student.batch_id` for batch-wise allocation rules.
4. **Fees**: Link with `academic_procedures` for fee payment verification.
5. **Warden**: Use `HoldsDesignation` to identify warden for approvals.

---

