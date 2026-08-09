# Visitor Hostel Module - Integration Guide

**What this module is:** The Visitor Hostel module will manage the institute's guest house - room bookings, visitor registration, billing, and accommodation for official guests, conference attendees, and visiting faculty.

**Why we need to build this:** Institutes host visitors who need accommodation. Currently booking is via phone/email, room availability is checked manually. This module will provide online room booking, automated availability checks, and digital billing.

**Why integration is needed:** Bookings are made by institute members (ExtraInfo/Faculty/Staff) for their guests. The host's identity and department need verification. Some bookings may be charged to project accounts (RSPC). The module needs organizational data from core modules.

**Key dependencies:** ExtraInfo, Faculty, Staff, Discipline

---

## Module Overview
The Visitor Hostel module manages guest accommodation booking, room allocation, billing, and visitor registration for the institute's guest house facilities.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Discipline`
| Column | Type | Description | Usage in Visitor Hostel |
|--------|------|-------------|-------------------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Discipline name | Booking department |

---

### 2. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Visitor Hostel |
|--------|------|-------------|-------------------------|
| `id` | CharField(20) | User ID | **Booking requestor** |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | **faculty/staff filter** |
| `phone_no` | BigIntegerField | Phone | Contact |
| `department` | ForeignKey(DepartmentInfo) | Department | **Booking department** |

#### Table: `Faculty`
| Column | Type | Description | Usage in Visitor Hostel |
|--------|------|-------------|-------------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Booking intender** |

#### Table: `Staff`
| Column | Type | Description | Usage in Visitor Hostel |
|--------|------|-------------|-------------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Booking intender** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in Visitor Hostel |
|--------|------|-------------|-------------------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | **Billing department** |

#### Table: `Designation`
| Column | Type | Description | Usage in Visitor Hostel |
|--------|------|-------------|-------------------------|
| `id` | AutoField | Primary Key | Designation reference |
| `name` | CharField(50) | Designation | **Approval authority** |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in Visitor Hostel |
|--------|------|-------------|-------------------------|
| `user` | ForeignKey(User) | User | Designation holder |
| `designation` | ForeignKey(Designation) | Designation | **VH Manager** |

---

## Data Models to Create in Visitor Hostel Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, Faculty, Staff, DepartmentInfo

# ==================== ROOM MANAGEMENT ====================

class Building(models.Model):
    """Guest house buildings"""
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=20, unique=True)
    location = models.CharField(max_length=200, blank=True)
    total_rooms = models.IntegerField(default=0)
    description = models.TextField(blank=True)
    contact_number = models.CharField(max_length=15, blank=True)
    is_active = models.BooleanField(default=True)

class RoomType(models.Model):
    """Types of rooms"""
    name = models.CharField(max_length=50)  # Single, Double, Suite, etc.
    description = models.TextField(blank=True)
    max_occupancy = models.IntegerField(default=1)
    has_ac = models.BooleanField(default=False)
    has_attached_bathroom = models.BooleanField(default=True)
    
    # Tariff
    tariff_per_day = models.DecimalField(max_digits=10, decimal_places=2)
    tariff_faculty = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    tariff_student = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    tariff_external = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)

class Room(models.Model):
    """Individual rooms"""
    STATUS = [
        ('AVAILABLE', 'Available'),
        ('OCCUPIED', 'Occupied'),
        ('MAINTENANCE', 'Under Maintenance'),
        ('BLOCKED', 'Blocked'),
    ]
    
    building = models.ForeignKey(Building, on_delete=models.CASCADE, related_name='rooms')
    room_number = models.CharField(max_length=20)
    room_type = models.ForeignKey(RoomType, on_delete=models.PROTECT)
    floor = models.IntegerField(default=0)
    
    status = models.CharField(max_length=20, choices=STATUS, default='AVAILABLE')
    
    # Amenities
    amenities = models.TextField(blank=True)  # JSON list
    
    # Capacity
    bed_count = models.IntegerField(default=1)
    
    class Meta:
        unique_together = ['building', 'room_number']
        ordering = ['building', 'room_number']

# ==================== BOOKING ====================

class BookingRequest(models.Model):
    """Room booking requests"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('CONFIRMED', 'Confirmed'),
        ('CHECKED_IN', 'Checked In'),
        ('CHECKED_OUT', 'Checked Out'),
        ('CANCELLED', 'Cancelled'),
        ('REJECTED', 'Rejected'),
    ]
    
    VISITOR_CATEGORY = [
        ('IIIT_FACULTY', 'IIIT Faculty'),
        ('IIIT_STAFF', 'IIIT Staff'),
        ('IIIT_STUDENT', 'IIIT Student'),
        ('GUEST_SPEAKER', 'Guest Speaker'),
        ('EXAMINER', 'External Examiner'),
        ('OFFICIAL', 'Official Visitor'),
        ('PERSONAL', 'Personal Guest'),
        ('OTHER', 'Other'),
    ]
    
    PURPOSE_TYPE = [
        ('ACADEMIC', 'Academic'),
        ('CONFERENCE', 'Conference/Seminar'),
        ('OFFICIAL', 'Official Work'),
        ('RECRUITMENT', 'Recruitment'),
        ('PERSONAL', 'Personal'),
        ('OTHER', 'Other'),
    ]
    
    # Booking details
    booking_number = models.CharField(max_length=50, unique=True)
    
    # Intender (who is booking)
    intender = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='vh_bookings')
    intender_department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True)
    
    # Visitor details
    visitor_category = models.CharField(max_length=20, choices=VISITOR_CATEGORY)
    visitor_name = models.CharField(max_length=200)
    visitor_organization = models.CharField(max_length=200, blank=True)
    visitor_designation = models.CharField(max_length=100, blank=True)
    visitor_phone = models.CharField(max_length=15)
    visitor_email = models.EmailField(blank=True)
    visitor_address = models.TextField(blank=True)
    
    # ID proof
    id_proof_type = models.CharField(max_length=50, blank=True)  # Aadhar, Passport, etc.
    id_proof_number = models.CharField(max_length=50, blank=True)
    
    # Number of visitors
    number_of_guests = models.IntegerField(default=1)
    number_of_rooms = models.IntegerField(default=1)
    room_type_preferred = models.ForeignKey(RoomType, on_delete=models.SET_NULL, null=True)
    
    # Stay details
    check_in_date = models.DateField()
    check_out_date = models.DateField()
    
    # Purpose
    purpose = models.CharField(max_length=20, choices=PURPOSE_TYPE)
    purpose_details = models.TextField(blank=True)
    
    # Billing
    billing_to = models.CharField(max_length=100, blank=True)  # Department, Project, Personal
    project_number = models.CharField(max_length=50, blank=True)  # If project-linked
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    # Approval
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='vh_approved')
    approved_at = models.DateTimeField(null=True, blank=True)
    approval_remarks = models.TextField(blank=True)
    
    # Timestamps
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    # Special requests
    special_requests = models.TextField(blank=True)

class RoomAllocation(models.Model):
    """Room allocations for bookings"""
    booking = models.ForeignKey(BookingRequest, on_delete=models.CASCADE, related_name='room_allocations')
    room = models.ForeignKey(Room, on_delete=models.PROTECT)
    
    guest_name = models.CharField(max_length=200)
    guest_phone = models.CharField(max_length=15, blank=True)
    
    check_in_date = models.DateField()
    check_out_date = models.DateField()
    actual_check_in = models.DateTimeField(null=True, blank=True)
    actual_check_out = models.DateTimeField(null=True, blank=True)
    
    # Tariff
    tariff_per_day = models.DecimalField(max_digits=10, decimal_places=2)
    
    class Meta:
        ordering = ['booking', 'room']

# ==================== CHECK-IN/CHECK-OUT ====================

class CheckInRecord(models.Model):
    """Check-in records"""
    allocation = models.OneToOneField(RoomAllocation, on_delete=models.CASCADE, related_name='check_in_record')
    
    check_in_time = models.DateTimeField()
    checked_in_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='vh_checkins')
    
    # Guest details at check-in
    guest_signature = models.ImageField(upload_to='vh/signatures/', blank=True)
    id_proof_copy = models.FileField(upload_to='vh/id_proofs/', blank=True)
    
    # Key handover
    keys_issued = models.BooleanField(default=True)
    key_number = models.CharField(max_length=20, blank=True)
    
    remarks = models.TextField(blank=True)

class CheckOutRecord(models.Model):
    """Check-out records"""
    allocation = models.OneToOneField(RoomAllocation, on_delete=models.CASCADE, related_name='check_out_record')
    
    check_out_time = models.DateTimeField()
    checked_out_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='vh_checkouts')
    
    # Key return
    keys_returned = models.BooleanField(default=False)
    
    # Room condition
    room_condition = models.CharField(max_length=50, blank=True)  # Good, Satisfactory, Damage
    damage_details = models.TextField(blank=True)
    
    remarks = models.TextField(blank=True)

# ==================== BILLING ====================

class Invoice(models.Model):
    """Invoices for bookings"""
    STATUS = [
        ('GENERATED', 'Generated'),
        ('PARTIALLY_PAID', 'Partially Paid'),
        ('PAID', 'Paid'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    PAYMENT_MODE = [
        ('CASH', 'Cash'),
        ('CARD', 'Card'),
        ('UPI', 'UPI'),
        ('CHEQUE', 'Cheque'),
        ('DEPARTMENT', 'Department Transfer'),
        ('PROJECT', 'Project Account'),
    ]
    
    booking = models.ForeignKey(BookingRequest, on_delete=models.CASCADE, related_name='invoices')
    invoice_number = models.CharField(max_length=50, unique=True)
    invoice_date = models.DateField()
    
    # Amounts
    room_charges = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    extra_charges = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    gst_amount = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    discount = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    
    # Payment
    amount_paid = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    balance_due = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    
    payment_mode = models.CharField(max_length=20, choices=PAYMENT_MODE, blank=True)
    payment_reference = models.CharField(max_length=100, blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='GENERATED')
    
    # Billing to
    billed_to_name = models.CharField(max_length=200)
    billed_to_address = models.TextField(blank=True)
    gstin = models.CharField(max_length=20, blank=True)
    
    generated_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)

class InvoiceItem(models.Model):
    """Line items in invoice"""
    invoice = models.ForeignKey(Invoice, on_delete=models.CASCADE, related_name='items')
    description = models.CharField(max_length=200)
    quantity = models.IntegerField(default=1)
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    amount = models.DecimalField(max_digits=12, decimal_places=2)

class Payment(models.Model):
    """Payment records"""
    invoice = models.ForeignKey(Invoice, on_delete=models.CASCADE, related_name='payments')
    amount = models.DecimalField(max_digits=12, decimal_places=2)
    payment_date = models.DateTimeField()
    payment_mode = models.CharField(max_length=20)
    reference_number = models.CharField(max_length=100, blank=True)
    received_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    remarks = models.TextField(blank=True)

# ==================== MEAL BOOKING ====================

class MealBooking(models.Model):
    """Meal bookings for guests"""
    MEAL_TYPE = [
        ('BREAKFAST', 'Breakfast'),
        ('LUNCH', 'Lunch'),
        ('DINNER', 'Dinner'),
        ('HIGH_TEA', 'High Tea'),
    ]
    
    booking = models.ForeignKey(BookingRequest, on_delete=models.CASCADE, related_name='meal_bookings')
    meal_date = models.DateField()
    meal_type = models.CharField(max_length=20, choices=MEAL_TYPE)
    number_of_persons = models.IntegerField(default=1)
    
    # Dietary preferences
    vegetarian = models.BooleanField(default=True)
    special_requirements = models.TextField(blank=True)
    
    # Billing
    rate_per_person = models.DecimalField(max_digits=8, decimal_places=2)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    
    class Meta:
        unique_together = ['booking', 'meal_date', 'meal_type']

# ==================== INVENTORY ====================

class RoomInventory(models.Model):
    """Room inventory items"""
    room = models.ForeignKey(Room, on_delete=models.CASCADE, related_name='inventory')
    item_name = models.CharField(max_length=100)
    quantity = models.IntegerField(default=1)
    condition = models.CharField(max_length=50, default='Good')
    last_checked = models.DateField(auto_now=True)

# ==================== FEEDBACK ====================

class GuestFeedback(models.Model):
    """Guest feedback"""
    booking = models.ForeignKey(BookingRequest, on_delete=models.CASCADE, related_name='feedback')
    
    # Ratings (1-5)
    room_cleanliness = models.IntegerField(null=True, blank=True)
    room_amenities = models.IntegerField(null=True, blank=True)
    staff_behavior = models.IntegerField(null=True, blank=True)
    food_quality = models.IntegerField(null=True, blank=True)
    overall_experience = models.IntegerField(null=True, blank=True)
    
    comments = models.TextField(blank=True)
    suggestions = models.TextField(blank=True)
    
    submitted_at = models.DateTimeField(auto_now_add=True)
```

---

## Integration Functions

### Get Intender Details
```python
def get_intender_details(user):
    """Get booking intender details"""
    from applications.globals.models import ExtraInfo
    
    extrainfo = ExtraInfo.objects.select_related('department', 'user').get(user=user)
    
    return {
        'id': extrainfo.id,
        'name': f"{user.first_name} {user.last_name}",
        'department': extrainfo.department,
        'phone': extrainfo.phone_no,
        'email': user.email,
    }
```

### Check Room Availability
```python
def check_room_availability(check_in, check_out, room_type=None, building=None):
    """Check available rooms for date range"""
    from django.db.models import Q
    
    # Get all rooms
    rooms = Room.objects.filter(status='AVAILABLE')
    
    if room_type:
        rooms = rooms.filter(room_type=room_type)
    if building:
        rooms = rooms.filter(building=building)
    
    # Exclude rooms with conflicting allocations
    conflicting = RoomAllocation.objects.filter(
        Q(check_in_date__lt=check_out) & Q(check_out_date__gt=check_in),
        booking__status__in=['CONFIRMED', 'CHECKED_IN']
    ).values_list('room_id', flat=True)
    
    available_rooms = rooms.exclude(id__in=conflicting)
    
    return available_rooms
```

### Calculate Tariff
```python
def calculate_tariff(room_type, visitor_category, num_days):
    """Calculate room tariff based on category"""
    if visitor_category in ['IIIT_FACULTY', 'IIIT_STAFF']:
        rate = room_type.tariff_faculty or room_type.tariff_per_day
    elif visitor_category == 'IIIT_STUDENT':
        rate = room_type.tariff_student or room_type.tariff_per_day
    else:
        rate = room_type.tariff_external or room_type.tariff_per_day
    
    return rate * num_days
```

### Generate Booking Number
```python
def generate_booking_number():
    """Generate unique booking number"""
    from datetime import date
    
    today = date.today()
    prefix = f"VH{today.strftime('%Y%m%d')}"
    
    last_booking = BookingRequest.objects.filter(
        booking_number__startswith=prefix
    ).order_by('-booking_number').first()
    
    if last_booking:
        last_seq = int(last_booking.booking_number[-4:])
        new_seq = last_seq + 1
    else:
        new_seq = 1
    
    return f"{prefix}{new_seq:04d}"
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/vh/api/rooms/` | GET | Room list |
| `/vh/api/rooms/availability/` | GET | Check availability |
| `/vh/api/bookings/` | GET, POST | Bookings |
| `/vh/api/bookings/<id>/approve/` | POST | Approve booking |
| `/vh/api/bookings/<id>/check-in/` | POST | Check-in |
| `/vh/api/bookings/<id>/check-out/` | POST | Check-out |
| `/vh/api/invoices/` | GET, POST | Invoices |
| `/vh/api/feedback/` | POST | Guest feedback |

---

## Important Sync Points

1. **Intender**: Faculty/Staff via `ExtraInfo` making the booking.
2. **Department**: `ExtraInfo.department` for billing purposes.
3. **Approval**: VH Manager via `HoldsDesignation`.
4. **Project Billing**: Link to RSPC/Project module for project-funded stays.

---

