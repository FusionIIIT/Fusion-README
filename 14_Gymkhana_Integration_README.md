# Gymkhana Module - Integration Guide

**What this module is:** The Gymkhana module will manage student extracurricular life - clubs (technical, cultural, sports), societies, events, budgets, inventory, elections for student council, and activity registrations.

**Why we need to build this:** Student life extends beyond academics but currently club management is manual - member lists on paper, budgets tracked in spreadsheets, event registrations via forms. This module will provide digital club management, automated elections, and centralized event coordination.

**Why integration is needed:** Club members are students (from Student table). Faculty advisors link to Faculty table. Election eligibility may require academic standing checks. The module needs identity data from production tables to know who can join what.

**Key dependencies:** Student, Batch, ExtraInfo, Faculty

---

## Module Overview
The Gymkhana module manages student clubs, societies, events, budgets, inventory, elections, and all extracurricular activities under the Gymkhana umbrella.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Batch`
| Column | Type | Description | Usage in Gymkhana |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Student batch |
| `name` | CharField(50) | Batch name | **Membership eligibility** |
| `year` | IntegerField | Year | Year filtering |

#### Table: `Discipline`
| Column | Type | Description | Usage in Gymkhana |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Branch reference |
| `name` | CharField(100) | Discipline name | Branch-wise statistics |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in Gymkhana |
|--------|------|-------------|-------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Club members** |
| `batch_id` | ForeignKey(Batch) | Batch | **Batch identification** |

---

### 3. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Gymkhana |
|--------|------|-------------|-------------------|
| `id` | CharField(20) | User ID | **Member ID** |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | student filter |
| `phone_no` | BigIntegerField | Phone | Contact |
| `profile_picture` | ImageField | Photo | Member photo |

#### Table: `Faculty`
| Column | Type | Description | Usage in Gymkhana |
|--------|------|-------------|-------------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Club faculty advisor** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in Gymkhana |
|--------|------|-------------|-------------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | Faculty department |

---

## Data Models to Create in Gymkhana Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, Faculty, DepartmentInfo
from applications.academic_information.models import Student
from applications.programme_curriculum.models import Batch

# ==================== CLUB MANAGEMENT ====================

class ClubCategory(models.Model):
    """Categories of clubs"""
    CATEGORY_TYPE = [
        ('CULTURAL', 'Cultural'),
        ('TECHNICAL', 'Technical'),
        ('SPORTS', 'Sports'),
        ('LITERARY', 'Literary'),
        ('SOCIAL', 'Social Service'),
        ('OTHER', 'Other'),
    ]
    
    name = models.CharField(max_length=100)
    category_type = models.CharField(max_length=20, choices=CATEGORY_TYPE)
    description = models.TextField(blank=True)

class Club(models.Model):
    """Student clubs and societies"""
    STATUS = [
        ('ACTIVE', 'Active'),
        ('INACTIVE', 'Inactive'),
        ('SUSPENDED', 'Suspended'),
    ]
    
    name = models.CharField(max_length=200)
    short_name = models.CharField(max_length=50, blank=True)
    category = models.ForeignKey(ClubCategory, on_delete=models.PROTECT)
    
    description = models.TextField()
    objectives = models.TextField(blank=True)
    
    # Logo and branding
    logo = models.ImageField(upload_to='gymkhana/clubs/logos/', blank=True)
    
    # Faculty advisor
    faculty_advisor = models.ForeignKey(Faculty, on_delete=models.SET_NULL, null=True, blank=True, related_name='advised_clubs')
    
    # Contact
    email = models.EmailField(blank=True)
    social_media = models.TextField(blank=True)  # JSON: {instagram: "", facebook: ""}
    
    # Membership
    max_members = models.IntegerField(null=True, blank=True)
    membership_fee = models.DecimalField(max_digits=8, decimal_places=2, default=0)
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='ACTIVE')
    established_date = models.DateField(null=True, blank=True)
    
    # Budget allocation
    annual_budget = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    
    class Meta:
        ordering = ['category', 'name']

# ==================== CLUB POSITIONS ====================

class ClubPosition(models.Model):
    """Positions within clubs"""
    POSITION_TYPE = [
        ('CORE', 'Core Team'),
        ('COORDINATOR', 'Coordinator'),
        ('MEMBER', 'Member'),
        ('VOLUNTEER', 'Volunteer'),
    ]
    
    club = models.ForeignKey(Club, on_delete=models.CASCADE, related_name='positions')
    title = models.CharField(max_length=100)  # President, Secretary, etc.
    position_type = models.CharField(max_length=20, choices=POSITION_TYPE)
    
    responsibilities = models.TextField(blank=True)
    max_holders = models.IntegerField(default=1)
    
    class Meta:
        unique_together = ['club', 'title']

class ClubMembership(models.Model):
    """Club memberships"""
    STATUS = [
        ('ACTIVE', 'Active'),
        ('INACTIVE', 'Inactive'),
        ('ALUMNI', 'Alumni'),
    ]
    
    club = models.ForeignKey(Club, on_delete=models.CASCADE, related_name='memberships')
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='club_memberships')
    position = models.ForeignKey(ClubPosition, on_delete=models.SET_NULL, null=True, blank=True)
    
    joined_date = models.DateField()
    ended_date = models.DateField(null=True, blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='ACTIVE')
    
    # For elected positions
    term_start = models.DateField(null=True, blank=True)
    term_end = models.DateField(null=True, blank=True)
    
    class Meta:
        unique_together = ['club', 'student', 'position']

# ==================== EVENTS ====================

class AcademicYear(models.Model):
    """Academic years for events"""
    name = models.CharField(max_length=20, unique=True)  # e.g., "2024-25"
    start_date = models.DateField()
    end_date = models.DateField()
    is_current = models.BooleanField(default=False)

class Event(models.Model):
    """Club events"""
    STATUS = [
        ('PROPOSED', 'Proposed'),
        ('APPROVED', 'Approved'),
        ('ONGOING', 'Ongoing'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    EVENT_TYPE = [
        ('WORKSHOP', 'Workshop'),
        ('COMPETITION', 'Competition'),
        ('SEMINAR', 'Seminar'),
        ('FEST', 'Fest'),
        ('MEETUP', 'Meetup'),
        ('PERFORMANCE', 'Performance'),
        ('HACKATHON', 'Hackathon'),
        ('SPORTS', 'Sports Event'),
        ('OTHER', 'Other'),
    ]
    
    name = models.CharField(max_length=200)
    club = models.ForeignKey(Club, on_delete=models.CASCADE, related_name='events')
    academic_year = models.ForeignKey(AcademicYear, on_delete=models.CASCADE)
    
    event_type = models.CharField(max_length=20, choices=EVENT_TYPE)
    description = models.TextField()
    
    # Date/Time
    start_datetime = models.DateTimeField()
    end_datetime = models.DateTimeField()
    
    # Venue
    venue = models.CharField(max_length=200)
    is_online = models.BooleanField(default=False)
    online_link = models.URLField(blank=True)
    
    # Participation
    max_participants = models.IntegerField(null=True, blank=True)
    registration_deadline = models.DateTimeField(null=True, blank=True)
    registration_fee = models.DecimalField(max_digits=8, decimal_places=2, default=0)
    
    # Open to
    open_to_all = models.BooleanField(default=True)
    eligible_batches = models.ManyToManyField(Batch, blank=True)
    
    # Budget
    estimated_budget = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    actual_budget = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='PROPOSED')
    
    # Approval
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='approved_events')
    approved_at = models.DateTimeField(null=True, blank=True)
    
    # Media
    poster = models.ImageField(upload_to='gymkhana/events/posters/', blank=True)
    
    # Coordinator
    coordinator = models.ForeignKey(Student, on_delete=models.SET_NULL, null=True, related_name='coordinated_events')
    
    created_at = models.DateTimeField(auto_now_add=True)

class EventRegistration(models.Model):
    """Event registrations"""
    STATUS = [
        ('REGISTERED', 'Registered'),
        ('ATTENDED', 'Attended'),
        ('ABSENT', 'Absent'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name='registrations')
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='event_registrations')
    
    registered_at = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=20, choices=STATUS, default='REGISTERED')
    
    # For team events
    team_name = models.CharField(max_length=100, blank=True)
    team_members = models.TextField(blank=True)  # JSON list
    
    # Payment
    fee_paid = models.BooleanField(default=False)
    payment_reference = models.CharField(max_length=100, blank=True)
    
    # Certificate
    certificate_issued = models.BooleanField(default=False)
    
    class Meta:
        unique_together = ['event', 'student']

class EventResult(models.Model):
    """Event competition results"""
    POSITION = [
        ('1ST', 'First'),
        ('2ND', 'Second'),
        ('3RD', 'Third'),
        ('MENTION', 'Special Mention'),
        ('PARTICIPANT', 'Participant'),
    ]
    
    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name='results')
    registration = models.ForeignKey(EventRegistration, on_delete=models.CASCADE)
    
    position = models.CharField(max_length=20, choices=POSITION)
    prize_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    remarks = models.TextField(blank=True)

# ==================== BUDGET & FINANCE ====================

class BudgetAllocation(models.Model):
    """Annual budget allocation to clubs"""
    club = models.ForeignKey(Club, on_delete=models.CASCADE, related_name='budget_allocations')
    academic_year = models.ForeignKey(AcademicYear, on_delete=models.CASCADE)
    
    allocated_amount = models.DecimalField(max_digits=12, decimal_places=2)
    spent_amount = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    
    allocated_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    allocated_at = models.DateTimeField(auto_now_add=True)
    
    remarks = models.TextField(blank=True)
    
    class Meta:
        unique_together = ['club', 'academic_year']

class ExpenseRequest(models.Model):
    """Club expense requests"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('DISBURSED', 'Disbursed'),
    ]
    
    club = models.ForeignKey(Club, on_delete=models.CASCADE, related_name='expense_requests')
    event = models.ForeignKey(Event, on_delete=models.SET_NULL, null=True, blank=True)
    
    description = models.TextField()
    amount_requested = models.DecimalField(max_digits=10, decimal_places=2)
    
    # Quotations
    quotation1 = models.FileField(upload_to='gymkhana/quotations/', blank=True)
    quotation2 = models.FileField(upload_to='gymkhana/quotations/', blank=True)
    quotation3 = models.FileField(upload_to='gymkhana/quotations/', blank=True)
    
    requested_by = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='expense_requests')
    requested_at = models.DateTimeField(auto_now_add=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    # Approval
    approved_amount = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='approved_expenses')
    approved_at = models.DateTimeField(null=True, blank=True)
    approval_remarks = models.TextField(blank=True)

class ExpenseRecord(models.Model):
    """Actual expense records"""
    expense_request = models.ForeignKey(ExpenseRequest, on_delete=models.CASCADE, related_name='expense_records')
    
    vendor_name = models.CharField(max_length=200)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    payment_date = models.DateField()
    
    bill = models.FileField(upload_to='gymkhana/bills/')
    payment_mode = models.CharField(max_length=50)
    
    recorded_by = models.ForeignKey(Student, on_delete=models.SET_NULL, null=True)

# ==================== INVENTORY ====================

class InventoryItem(models.Model):
    """Club inventory items"""
    CONDITION = [
        ('GOOD', 'Good'),
        ('FAIR', 'Fair'),
        ('POOR', 'Poor'),
        ('DAMAGED', 'Damaged'),
    ]
    
    club = models.ForeignKey(Club, on_delete=models.CASCADE, related_name='inventory')
    
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    quantity = models.IntegerField(default=1)
    
    purchase_date = models.DateField(null=True, blank=True)
    purchase_cost = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    condition = models.CharField(max_length=20, choices=CONDITION, default='GOOD')
    location = models.CharField(max_length=200, blank=True)
    
    last_checked = models.DateField(null=True, blank=True)

class InventoryIssue(models.Model):
    """Inventory item issues/loans"""
    STATUS = [
        ('ISSUED', 'Issued'),
        ('RETURNED', 'Returned'),
        ('LOST', 'Lost'),
        ('DAMAGED', 'Damaged'),
    ]
    
    item = models.ForeignKey(InventoryItem, on_delete=models.CASCADE, related_name='issues')
    issued_to = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='inventory_issues')
    
    quantity = models.IntegerField(default=1)
    purpose = models.CharField(max_length=200)
    
    issued_date = models.DateField()
    expected_return = models.DateField()
    actual_return = models.DateField(null=True, blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='ISSUED')
    
    issued_by = models.ForeignKey(Student, on_delete=models.SET_NULL, null=True, related_name='inventory_issued')
    return_condition = models.CharField(max_length=100, blank=True)

# ==================== ELECTIONS ====================

class Election(models.Model):
    """Gymkhana/Club elections"""
    STATUS = [
        ('UPCOMING', 'Upcoming'),
        ('NOMINATIONS_OPEN', 'Nominations Open'),
        ('VOTING_OPEN', 'Voting Open'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    title = models.CharField(max_length=200)
    description = models.TextField()
    
    academic_year = models.ForeignKey(AcademicYear, on_delete=models.CASCADE)
    club = models.ForeignKey(Club, on_delete=models.CASCADE, null=True, blank=True)  # Null for central elections
    
    # Eligible voters
    eligible_batches = models.ManyToManyField(Batch)
    
    # Timeline
    nomination_start = models.DateTimeField()
    nomination_end = models.DateTimeField()
    voting_start = models.DateTimeField()
    voting_end = models.DateTimeField()
    
    status = models.CharField(max_length=20, choices=STATUS, default='UPCOMING')
    
    created_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)

class ElectionPosition(models.Model):
    """Positions in an election"""
    election = models.ForeignKey(Election, on_delete=models.CASCADE, related_name='election_positions')
    position_name = models.CharField(max_length=100)
    description = models.TextField(blank=True)
    
    # Eligibility
    eligible_batches = models.ManyToManyField(Batch, blank=True)

class Nomination(models.Model):
    """Election nominations"""
    STATUS = [
        ('SUBMITTED', 'Submitted'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
        ('WITHDRAWN', 'Withdrawn'),
    ]
    
    election_position = models.ForeignKey(ElectionPosition, on_delete=models.CASCADE, related_name='nominations')
    candidate = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='nominations')
    
    manifesto = models.TextField()
    manifesto_file = models.FileField(upload_to='gymkhana/elections/manifesto/', blank=True)
    
    submitted_at = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=20, choices=STATUS, default='SUBMITTED')
    
    # Approval
    verified_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    verification_remarks = models.TextField(blank=True)

class Vote(models.Model):
    """Election votes (anonymous)"""
    election_position = models.ForeignKey(ElectionPosition, on_delete=models.CASCADE, related_name='votes')
    nomination = models.ForeignKey(Nomination, on_delete=models.CASCADE, related_name='votes')
    
    voted_at = models.DateTimeField(auto_now_add=True)
    
    # Note: Voter identity not stored to maintain anonymity
    # Use separate model to track who has voted

class VoterRecord(models.Model):
    """Track who has voted (not how)"""
    election = models.ForeignKey(Election, on_delete=models.CASCADE, related_name='voter_records')
    voter = models.ForeignKey(Student, on_delete=models.CASCADE)
    voted_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['election', 'voter']

# ==================== ACHIEVEMENTS ====================

class Achievement(models.Model):
    """Student achievements"""
    LEVEL = [
        ('INSTITUTE', 'Institute'),
        ('STATE', 'State'),
        ('NATIONAL', 'National'),
        ('INTERNATIONAL', 'International'),
    ]
    
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='achievements')
    club = models.ForeignKey(Club, on_delete=models.SET_NULL, null=True, blank=True)
    
    title = models.CharField(max_length=200)
    description = models.TextField()
    
    level = models.CharField(max_length=20, choices=LEVEL)
    event_name = models.CharField(max_length=200)
    organizer = models.CharField(max_length=200, blank=True)
    
    achievement_date = models.DateField()
    
    # Recognition
    position = models.CharField(max_length=50, blank=True)  # 1st, Gold Medal, etc.
    prize = models.CharField(max_length=200, blank=True)
    
    # Proof
    certificate = models.FileField(upload_to='gymkhana/achievements/', blank=True)
    
    verified = models.BooleanField(default=False)
    verified_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
```

---

## Integration Functions

### Get Club Members
```python
def get_club_members(club, active_only=True):
    """Get all members of a club"""
    from applications.academic_information.models import Student
    
    memberships = ClubMembership.objects.filter(club=club)
    
    if active_only:
        memberships = memberships.filter(status='ACTIVE')
    
    return memberships.select_related('student', 'student__id', 'student__id__user', 'position')
```

### Check Election Eligibility
```python
def check_election_eligibility(student, election_position):
    """Check if student is eligible for election position"""
    
    # Check batch eligibility
    if election_position.eligible_batches.exists():
        if student.batch_id not in election_position.eligible_batches.all():
            return False, "Batch not eligible"
    
    return True, "Eligible"
```

### Get Student Info for Membership
```python
def get_student_for_club(user):
    """Get student details for club membership"""
    from applications.globals.models import ExtraInfo
    from applications.academic_information.models import Student
    
    extrainfo = ExtraInfo.objects.get(user=user)
    student = Student.objects.select_related('batch_id').get(id=extrainfo)
    
    return {
        'student': student,
        'roll_no': extrainfo.id,
        'name': f"{user.first_name} {user.last_name}",
        'batch': student.batch_id,
        'phone': extrainfo.phone_no,
        'email': user.email,
    }
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/gymkhana/api/clubs/` | GET | Club list |
| `/gymkhana/api/clubs/<id>/members/` | GET | Club members |
| `/gymkhana/api/clubs/<id>/join/` | POST | Join club |
| `/gymkhana/api/events/` | GET | Events |
| `/gymkhana/api/events/<id>/register/` | POST | Register for event |
| `/gymkhana/api/elections/` | GET | Elections |
| `/gymkhana/api/elections/<id>/nominate/` | POST | File nomination |
| `/gymkhana/api/elections/<id>/vote/` | POST | Cast vote |

---

## Important Sync Points

1. **Member Base**: Use `Student` model for all club members.
2. **Batch**: Get from `Student.batch_id` for eligibility checks.
3. **Faculty Advisor**: Link to `Faculty` model.
4. **Contact Info**: Get from `ExtraInfo` (phone, profile picture).

---

