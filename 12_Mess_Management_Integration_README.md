# Mess Management Module - Integration Guide

**What this module is:** The Mess Management module will handle dining operations - menu planning, student mess registration, meal tracking, billing, rebates for missed meals, feedback collection, and mess worker management.

**Why we need to build this:** Hostelers eat in messes. Currently mess registration is manual, rebate requests on paper. This module will digitize mess operations - online registration, automated billing, digital rebate workflows.

**Why integration is needed:** Mess members are students (from Student table). Mess dues sync with the Dues table in academic_procedures. When students apply for rebates during hostel leave, it links to Hostel module. Student identity and dues data come from core modules.

**Key dependencies:** Student, Batch, ExtraInfo, Dues (mess_due field)

---

## Module Overview
The Mess Management module handles mess menu planning, student mess registration, meal tracking, billing, feedback, rebates, and all mess-related administrative functions.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Batch`
| Column | Type | Description | Usage in Mess |
|--------|------|-------------|---------------|
| `id` | AutoField | Primary Key | Student batch |
| `name` | CharField(50) | Batch name | Batch-wise billing |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in Mess |
|--------|------|-------------|---------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Mess member** |
| `batch_id` | ForeignKey(Batch) | Batch | Batch info |

---

### 3. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Mess |
|--------|------|-------------|---------------|
| `id` | CharField(20) | User ID | **Member ID** |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | student/faculty/staff |
| `phone_no` | BigIntegerField | Phone | Contact |

#### Table: `Faculty`
| Column | Type | Description | Usage in Mess |
|--------|------|-------------|---------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | Faculty mess member |

#### Table: `Staff`
| Column | Type | Description | Usage in Mess |
|--------|------|-------------|---------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | Staff mess member |

---

## Data Models to Create in Mess Management Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo
from applications.academic_information.models import Student
from applications.programme_curriculum.models import Batch

# ==================== MESS CONFIGURATION ====================

class Mess(models.Model):
    """Mess facilities"""
    MESS_TYPE = [
        ('VEG', 'Vegetarian'),
        ('NON_VEG', 'Non-Vegetarian'),
        ('BOTH', 'Both'),
    ]
    
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=20, unique=True)
    location = models.CharField(max_length=200, blank=True)
    
    mess_type = models.CharField(max_length=20, choices=MESS_TYPE, default='BOTH')
    
    # Capacity
    seating_capacity = models.IntegerField(default=500)
    
    # Timings
    breakfast_start = models.TimeField(null=True, blank=True)
    breakfast_end = models.TimeField(null=True, blank=True)
    lunch_start = models.TimeField(null=True, blank=True)
    lunch_end = models.TimeField(null=True, blank=True)
    dinner_start = models.TimeField(null=True, blank=True)
    dinner_end = models.TimeField(null=True, blank=True)
    
    # Contact
    contact_number = models.CharField(max_length=15, blank=True)
    
    # Management
    mess_incharge = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='mess_incharge_of')
    mess_warden = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='mess_warden_of')
    
    # Contractor
    contractor_name = models.CharField(max_length=200, blank=True)
    contractor_contact = models.CharField(max_length=15, blank=True)
    contract_start = models.DateField(null=True, blank=True)
    contract_end = models.DateField(null=True, blank=True)
    
    is_active = models.BooleanField(default=True)

# ==================== MENU MANAGEMENT ====================

class MenuItem(models.Model):
    """Menu items"""
    CATEGORY = [
        ('BREAKFAST', 'Breakfast'),
        ('LUNCH', 'Lunch'),
        ('DINNER', 'Dinner'),
        ('SNACKS', 'Snacks'),
        ('BEVERAGES', 'Beverages'),
    ]
    
    TYPE = [
        ('VEG', 'Vegetarian'),
        ('NON_VEG', 'Non-Vegetarian'),
    ]
    
    name = models.CharField(max_length=100)
    category = models.CharField(max_length=20, choices=CATEGORY)
    item_type = models.CharField(max_length=20, choices=TYPE, default='VEG')
    description = models.TextField(blank=True)
    
    # Nutritional info
    calories = models.IntegerField(null=True, blank=True)
    protein = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)
    
    is_active = models.BooleanField(default=True)

class MessMenu(models.Model):
    """Weekly menu schedule"""
    DAY_CHOICES = [
        ('MON', 'Monday'),
        ('TUE', 'Tuesday'),
        ('WED', 'Wednesday'),
        ('THU', 'Thursday'),
        ('FRI', 'Friday'),
        ('SAT', 'Saturday'),
        ('SUN', 'Sunday'),
    ]
    
    MEAL_TYPE = [
        ('BREAKFAST', 'Breakfast'),
        ('LUNCH', 'Lunch'),
        ('DINNER', 'Dinner'),
    ]
    
    mess = models.ForeignKey(Mess, on_delete=models.CASCADE, related_name='menus')
    day = models.CharField(max_length=3, choices=DAY_CHOICES)
    meal_type = models.CharField(max_length=20, choices=MEAL_TYPE)
    
    # Menu items
    items = models.ManyToManyField(MenuItem, related_name='menu_schedules')
    
    # Validity
    effective_from = models.DateField()
    effective_until = models.DateField(null=True, blank=True)
    
    class Meta:
        unique_together = ['mess', 'day', 'meal_type', 'effective_from']

class SpecialMenu(models.Model):
    """Special menus for events/festivals"""
    mess = models.ForeignKey(Mess, on_delete=models.CASCADE, related_name='special_menus')
    date = models.DateField()
    occasion = models.CharField(max_length=200)
    
    breakfast_items = models.TextField(blank=True)
    lunch_items = models.TextField(blank=True)
    dinner_items = models.TextField(blank=True)
    
    created_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)

# ==================== MEMBER REGISTRATION ====================

class MessRegistration(models.Model):
    """Mess member registration"""
    STATUS = [
        ('ACTIVE', 'Active'),
        ('INACTIVE', 'Inactive'),
        ('SUSPENDED', 'Suspended'),
    ]
    
    PREFERENCE = [
        ('VEG', 'Vegetarian'),
        ('NON_VEG', 'Non-Vegetarian'),
    ]
    
    # Member (can be student, faculty, or staff)
    member = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='mess_registrations')
    mess = models.ForeignKey(Mess, on_delete=models.CASCADE)
    
    # Registration details
    registration_date = models.DateField()
    valid_from = models.DateField()
    valid_until = models.DateField(null=True, blank=True)
    
    # Preference
    food_preference = models.CharField(max_length=20, choices=PREFERENCE, default='VEG')
    
    status = models.CharField(max_length=20, choices=STATUS, default='ACTIVE')
    
    class Meta:
        ordering = ['-registration_date']

class MessCard(models.Model):
    """Mess cards for members"""
    registration = models.OneToOneField(MessRegistration, on_delete=models.CASCADE, related_name='card')
    card_number = models.CharField(max_length=50, unique=True)
    issued_date = models.DateField()
    
    is_active = models.BooleanField(default=True)
    
    # For lost/replacement
    replacement_count = models.IntegerField(default=0)

# ==================== MEAL TRACKING ====================

class MealEntry(models.Model):
    """Individual meal entries"""
    MEAL_TYPE = [
        ('BREAKFAST', 'Breakfast'),
        ('LUNCH', 'Lunch'),
        ('DINNER', 'Dinner'),
    ]
    
    registration = models.ForeignKey(MessRegistration, on_delete=models.CASCADE, related_name='meal_entries')
    date = models.DateField()
    meal_type = models.CharField(max_length=20, choices=MEAL_TYPE)
    
    entry_time = models.TimeField()
    
    # Verification
    card_number = models.CharField(max_length=50, blank=True)
    verified_by = models.CharField(max_length=100, blank=True)  # Terminal/Staff ID
    
    class Meta:
        unique_together = ['registration', 'date', 'meal_type']
        ordering = ['-date', '-entry_time']

class GuestMealEntry(models.Model):
    """Guest meal entries"""
    MEAL_TYPE = [
        ('BREAKFAST', 'Breakfast'),
        ('LUNCH', 'Lunch'),
        ('DINNER', 'Dinner'),
    ]
    
    mess = models.ForeignKey(Mess, on_delete=models.CASCADE)
    date = models.DateField()
    meal_type = models.CharField(max_length=20, choices=MEAL_TYPE)
    
    # Host
    host = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='guest_meals')
    
    # Guest details
    guest_count = models.IntegerField(default=1)
    guest_names = models.TextField(blank=True)
    
    # Billing
    rate = models.DecimalField(max_digits=8, decimal_places=2)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    
    entry_time = models.TimeField()

# ==================== BILLING ====================

class MessBillPeriod(models.Model):
    """Billing periods"""
    name = models.CharField(max_length=50)  # e.g., "January 2024"
    mess = models.ForeignKey(Mess, on_delete=models.CASCADE)
    
    start_date = models.DateField()
    end_date = models.DateField()
    
    # Rates
    per_day_rate = models.DecimalField(max_digits=8, decimal_places=2)
    breakfast_rate = models.DecimalField(max_digits=8, decimal_places=2, null=True, blank=True)
    lunch_rate = models.DecimalField(max_digits=8, decimal_places=2, null=True, blank=True)
    dinner_rate = models.DecimalField(max_digits=8, decimal_places=2, null=True, blank=True)
    
    is_finalized = models.BooleanField(default=False)
    
    class Meta:
        unique_together = ['mess', 'start_date', 'end_date']

class MessBill(models.Model):
    """Monthly mess bills"""
    STATUS = [
        ('GENERATED', 'Generated'),
        ('PAID', 'Paid'),
        ('PARTIAL', 'Partially Paid'),
        ('OVERDUE', 'Overdue'),
    ]
    
    registration = models.ForeignKey(MessRegistration, on_delete=models.CASCADE, related_name='bills')
    billing_period = models.ForeignKey(MessBillPeriod, on_delete=models.CASCADE)
    
    # Bill details
    bill_number = models.CharField(max_length=50, unique=True)
    bill_date = models.DateField()
    
    # Days/Meals
    total_days = models.IntegerField(default=0)
    rebate_days = models.IntegerField(default=0)
    chargeable_days = models.IntegerField(default=0)
    
    # Amounts
    basic_charges = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    extra_charges = models.DecimalField(max_digits=10, decimal_places=2, default=0)  # Guest meals, etc.
    rebate_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    fine_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    
    # Payment
    amount_paid = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    balance_due = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    status = models.CharField(max_length=20, choices=STATUS, default='GENERATED')
    due_date = models.DateField()
    
    class Meta:
        unique_together = ['registration', 'billing_period']

class MessPayment(models.Model):
    """Payment records"""
    PAYMENT_MODE = [
        ('CASH', 'Cash'),
        ('CARD', 'Card'),
        ('UPI', 'UPI'),
        ('BANK_TRANSFER', 'Bank Transfer'),
        ('FEE_DEDUCTION', 'Fee Account Deduction'),
    ]
    
    bill = models.ForeignKey(MessBill, on_delete=models.CASCADE, related_name='payments')
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    payment_date = models.DateTimeField()
    payment_mode = models.CharField(max_length=20, choices=PAYMENT_MODE)
    transaction_id = models.CharField(max_length=100, blank=True)
    
    received_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    remarks = models.TextField(blank=True)

# ==================== REBATE MANAGEMENT ====================

class RebateApplication(models.Model):
    """Mess rebate applications"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    REASON = [
        ('VACATION', 'Vacation'),
        ('MEDICAL', 'Medical'),
        ('INTERNSHIP', 'Internship'),
        ('CONFERENCE', 'Conference/Seminar'),
        ('OTHER', 'Other'),
    ]
    
    registration = models.ForeignKey(MessRegistration, on_delete=models.CASCADE, related_name='rebate_applications')
    
    from_date = models.DateField()
    to_date = models.DateField()
    total_days = models.IntegerField()
    
    reason = models.CharField(max_length=20, choices=REASON)
    reason_details = models.TextField(blank=True)
    
    # Documentation
    supporting_document = models.FileField(upload_to='mess/rebates/', blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    # Approval
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='approved_rebates')
    approved_at = models.DateTimeField(null=True, blank=True)
    approval_remarks = models.TextField(blank=True)
    
    # Rebate calculation
    rebate_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    applied_at = models.DateTimeField(auto_now_add=True)

# ==================== FEEDBACK ====================

class MessFeedback(models.Model):
    """Mess feedback"""
    registration = models.ForeignKey(MessRegistration, on_delete=models.CASCADE, related_name='feedbacks')
    date = models.DateField()
    meal_type = models.CharField(max_length=20)
    
    # Ratings (1-5)
    food_quality = models.IntegerField()
    quantity = models.IntegerField()
    hygiene = models.IntegerField()
    service = models.IntegerField()
    overall = models.IntegerField()
    
    comments = models.TextField(blank=True)
    suggestions = models.TextField(blank=True)
    
    submitted_at = models.DateTimeField(auto_now_add=True)

class MessComplaint(models.Model):
    """Mess complaints"""
    STATUS = [
        ('OPEN', 'Open'),
        ('IN_PROGRESS', 'In Progress'),
        ('RESOLVED', 'Resolved'),
        ('CLOSED', 'Closed'),
    ]
    
    CATEGORY = [
        ('FOOD_QUALITY', 'Food Quality'),
        ('HYGIENE', 'Hygiene'),
        ('SERVICE', 'Service'),
        ('TIMING', 'Timing'),
        ('QUANTITY', 'Quantity'),
        ('OTHER', 'Other'),
    ]
    
    registration = models.ForeignKey(MessRegistration, on_delete=models.CASCADE, related_name='complaints')
    
    category = models.CharField(max_length=20, choices=CATEGORY)
    subject = models.CharField(max_length=200)
    description = models.TextField()
    
    # Evidence
    image = models.ImageField(upload_to='mess/complaints/', blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='OPEN')
    
    # Resolution
    resolved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    resolution = models.TextField(blank=True)
    resolved_at = models.DateTimeField(null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)

# ==================== INVENTORY (BASIC) ====================

class MessInventoryItem(models.Model):
    """Mess inventory items"""
    mess = models.ForeignKey(Mess, on_delete=models.CASCADE, related_name='inventory')
    name = models.CharField(max_length=100)
    category = models.CharField(max_length=50)  # Groceries, Vegetables, Dairy, etc.
    unit = models.CharField(max_length=20)  # kg, ltr, pcs, etc.
    
    current_stock = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    minimum_stock = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    last_updated = models.DateTimeField(auto_now=True)

# ==================== MESS COMMITTEE ====================

class MessCommitteeMember(models.Model):
    """Mess committee members"""
    ROLE = [
        ('CONVENER', 'Convener'),
        ('MEMBER', 'Member'),
        ('SECRETARY', 'Secretary'),
    ]
    
    mess = models.ForeignKey(Mess, on_delete=models.CASCADE, related_name='committee')
    member = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE)
    role = models.CharField(max_length=20, choices=ROLE, default='MEMBER')
    
    # Term
    from_date = models.DateField()
    to_date = models.DateField(null=True, blank=True)
    
    is_active = models.BooleanField(default=True)
```

---

## Integration Functions

### Register Student for Mess
```python
def register_student_for_mess(student, mess, food_preference='VEG'):
    """Register a student for mess"""
    from datetime import date
    from applications.globals.models import ExtraInfo
    
    # Get ExtraInfo
    extrainfo = student.id  # Student.id is OneToOneField to ExtraInfo
    
    registration = MessRegistration.objects.create(
        member=extrainfo,
        mess=mess,
        registration_date=date.today(),
        valid_from=date.today(),
        food_preference=food_preference,
        status='ACTIVE'
    )
    
    return registration
```

### Get Student's Current Mess
```python
def get_student_mess(student):
    """Get student's current mess registration"""
    from datetime import date
    
    registration = MessRegistration.objects.filter(
        member=student.id,
        status='ACTIVE',
        valid_from__lte=date.today()
    ).first()
    
    return registration
```

### Calculate Monthly Bill
```python
def calculate_monthly_bill(registration, billing_period):
    """Calculate mess bill for a member"""
    from django.db.models import Count
    
    # Count meals taken
    meal_count = MealEntry.objects.filter(
        registration=registration,
        date__gte=billing_period.start_date,
        date__lte=billing_period.end_date
    ).count()
    
    # Calculate rebate days
    rebate_days = RebateApplication.objects.filter(
        registration=registration,
        status='APPROVED',
        from_date__lte=billing_period.end_date,
        to_date__gte=billing_period.start_date
    ).aggregate(
        total=models.Sum('total_days')
    )['total'] or 0
    
    # Calculate
    total_days = (billing_period.end_date - billing_period.start_date).days + 1
    chargeable_days = total_days - rebate_days
    basic_charges = chargeable_days * billing_period.per_day_rate
    
    return {
        'total_days': total_days,
        'rebate_days': rebate_days,
        'chargeable_days': chargeable_days,
        'basic_charges': basic_charges,
    }
```

### Sync with Fee Module
```python
def sync_mess_dues_to_fees(student):
    """Sync pending mess dues to academic fees module"""
    from applications.academic_procedures.models import Dues
    
    # Get pending mess bills
    pending_bills = MessBill.objects.filter(
        registration__member=student.id,
        status__in=['GENERATED', 'PARTIAL', 'OVERDUE']
    ).aggregate(total=models.Sum('balance_due'))
    
    # Update mess dues in Dues table
    if pending_bills['total']:
        dues, created = Dues.objects.get_or_create(
            student_id=student,
            defaults={
                'mess_due': pending_bills['total'],
                'hostel_due': 0,
                'library_due': 0,
                'placement_cell_due': 0,
                'academic_due': 0
            }
        )
        if not created:
            dues.mess_due = pending_bills['total']
            dues.save()
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/mess/api/messes/` | GET | Mess list |
| `/mess/api/menu/` | GET | Current menu |
| `/mess/api/registrations/` | GET, POST | Member registrations |
| `/mess/api/meals/` | POST | Record meal entry |
| `/mess/api/bills/` | GET | View bills |
| `/mess/api/rebates/` | GET, POST | Rebate applications |
| `/mess/api/feedback/` | POST | Submit feedback |
| `/mess/api/complaints/` | GET, POST | Complaints |

---

## Important Sync Points

1. **Member Base**: Use `ExtraInfo` for all member types (student/faculty/staff).
2. **Student Info**: Get from `Student` model for student-specific operations.
3. **Fee Integration**: Sync pending mess dues with `academic_procedures.Dues`.
4. **Hostel Integration**: Coordinate with Hostel Management for leave-based rebates.

---

