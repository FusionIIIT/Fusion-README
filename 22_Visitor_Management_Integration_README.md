# Visitor Management Module - Integration Guide

**What this module is:** The Visitor Management module will handle campus visitor registration, gate passes, entry/exit tracking, host verification, and visitor logs.

**Why we need to build this:** Campus security requires visitor tracking but currently it's manual - visitors sign a register, hosts are called for verification. This module will provide digital visitor registration, automated host notification, and real-time entry/exit tracking.

**Why integration is needed:** Hosts are faculty/staff (from ExtraInfo/Faculty/Staff). Visitors may visit specific departments (DepartmentInfo). The module needs to verify host identity and organizational affiliation from production tables.

**Key dependencies:** ExtraInfo, Faculty, Staff, DepartmentInfo

---

## Module Overview
Manages visitor registration, gate passes, and visitor tracking.

## Core Dependencies
- `globals.ExtraInfo` - Host details
- `globals.Faculty` - Faculty hosts
- `globals.DepartmentInfo` - Department for visits

---

## Key Models

```python
from django.db import models
from applications.globals.models import ExtraInfo, DepartmentInfo

class Visitor(models.Model):
    """Visitor records"""
    PURPOSE = [
        ('OFFICIAL', 'Official'),
        ('PERSONAL', 'Personal'),
        ('VENDOR', 'Vendor'),
        ('INTERVIEW', 'Interview'),
        ('EVENT', 'Event'),
        ('OTHER', 'Other'),
    ]
    
    # Visitor details
    name = models.CharField(max_length=200)
    phone = models.CharField(max_length=15)
    email = models.EmailField(blank=True)
    organization = models.CharField(max_length=200, blank=True)
    
    # ID proof
    id_type = models.CharField(max_length=50, blank=True)
    id_number = models.CharField(max_length=50, blank=True)
    photo = models.ImageField(upload_to='visitors/photos/', blank=True)
    
    # Visit details
    purpose = models.CharField(max_length=20, choices=PURPOSE)
    purpose_details = models.TextField(blank=True)
    
    # Host
    host = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE)
    department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True)
    
    # Timing
    expected_arrival = models.DateTimeField()
    expected_departure = models.DateTimeField(null=True)
    
    actual_arrival = models.DateTimeField(null=True)
    actual_departure = models.DateTimeField(null=True)
    
    # Pass
    pass_number = models.CharField(max_length=50, unique=True)
    pass_valid_until = models.DateTimeField()
    
    # Status
    is_checked_in = models.BooleanField(default=False)
    is_checked_out = models.BooleanField(default=False)
    
    # Approval
    is_pre_approved = models.BooleanField(default=False)
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='approved_visitors')
    
    created_at = models.DateTimeField(auto_now_add=True)

class VehiclePass(models.Model):
    """Vehicle gate passes"""
    visitor = models.ForeignKey(Visitor, on_delete=models.CASCADE, related_name='vehicles', null=True)
    
    vehicle_type = models.CharField(max_length=50)  # Car, Bike, etc.
    vehicle_number = models.CharField(max_length=20)
    
    entry_time = models.DateTimeField(null=True)
    exit_time = models.DateTimeField(null=True)
    
    parking_slot = models.CharField(max_length=20, blank=True)

class VisitorBlacklist(models.Model):
    """Blacklisted visitors"""
    name = models.CharField(max_length=200)
    id_number = models.CharField(max_length=50, blank=True)
    phone = models.CharField(max_length=15, blank=True)
    
    reason = models.TextField()
    
    blacklisted_at = models.DateTimeField(auto_now_add=True)
    blacklisted_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    
    is_permanent = models.BooleanField(default=False)
    valid_until = models.DateField(null=True)
```

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `Visitor` | Visitor records |
| `VehiclePass` | Vehicle gate passes |
| `VisitorBlacklist` | Blacklisted visitors |

---

## Integration with globals

### Get Host Details
```python
def get_host_details(host_id):
    """Get host details from globals"""
    from applications.globals.models import ExtraInfo
    
    host = ExtraInfo.objects.select_related('user', 'department').get(id=host_id)
    
    return {
        'id': host.id,
        'name': f"{host.user.first_name} {host.user.last_name}",
        'email': host.user.email,
        'department': host.department.name if host.department else None,
        'user_type': host.user_type,
    }
```

### Get Department for Visit
```python
def get_department_info(department_id):
    """Get department info for visitor pass"""
    from applications.globals.models import DepartmentInfo
    
    dept = DepartmentInfo.objects.get(id=department_id)
    return {
        'id': dept.id,
        'name': dept.name,
    }
```

### Search for Host
```python
def search_host(query):
    """Search for host by name or ID"""
    from applications.globals.models import ExtraInfo
    from django.db.models import Q
    
    return ExtraInfo.objects.filter(
        Q(user__first_name__icontains=query) |
        Q(user__last_name__icontains=query) |
        Q(id__icontains=query)
    ).select_related('user', 'department')[:10]
```

---

## Common Operations

### Register Visitor
```python
def register_visitor(visitor_data, host_id):
    """Register a new visitor"""
    from applications.globals.models import ExtraInfo
    from django.utils import timezone
    import uuid
    
    host = ExtraInfo.objects.get(id=host_id)
    
    # Check blacklist
    if is_blacklisted(visitor_data.get('id_number'), visitor_data.get('phone')):
        raise ValueError("Visitor is blacklisted")
    
    # Generate pass number
    pass_number = f"VP-{timezone.now().strftime('%Y%m%d')}-{uuid.uuid4().hex[:6].upper()}"
    
    visitor = Visitor.objects.create(
        name=visitor_data['name'],
        phone=visitor_data['phone'],
        email=visitor_data.get('email', ''),
        organization=visitor_data.get('organization', ''),
        id_type=visitor_data.get('id_type', ''),
        id_number=visitor_data.get('id_number', ''),
        purpose=visitor_data['purpose'],
        purpose_details=visitor_data.get('purpose_details', ''),
        host=host,
        department=host.department,
        expected_arrival=visitor_data['expected_arrival'],
        expected_departure=visitor_data.get('expected_departure'),
        pass_number=pass_number,
        pass_valid_until=visitor_data.get('pass_valid_until', timezone.now() + timezone.timedelta(hours=8)),
    )
    
    return visitor
```

### Check-In Visitor
```python
def check_in_visitor(pass_number):
    """Check in a visitor"""
    from django.utils import timezone
    
    visitor = Visitor.objects.get(pass_number=pass_number)
    
    if visitor.is_checked_in:
        raise ValueError("Visitor already checked in")
    
    if timezone.now() > visitor.pass_valid_until:
        raise ValueError("Pass has expired")
    
    visitor.is_checked_in = True
    visitor.actual_arrival = timezone.now()
    visitor.save()
    
    return visitor
```

### Check-Out Visitor
```python
def check_out_visitor(pass_number):
    """Check out a visitor"""
    from django.utils import timezone
    
    visitor = Visitor.objects.get(pass_number=pass_number)
    
    if not visitor.is_checked_in:
        raise ValueError("Visitor not checked in")
    
    if visitor.is_checked_out:
        raise ValueError("Visitor already checked out")
    
    visitor.is_checked_out = True
    visitor.actual_departure = timezone.now()
    visitor.save()
    
    # Check out vehicles
    visitor.vehicles.filter(exit_time__isnull=True).update(exit_time=timezone.now())
    
    return visitor
```

### Check Blacklist
```python
def is_blacklisted(id_number=None, phone=None):
    """Check if visitor is blacklisted"""
    from django.utils import timezone
    from django.db.models import Q
    
    query = Q()
    if id_number:
        query |= Q(id_number=id_number)
    if phone:
        query |= Q(phone=phone)
    
    if not query:
        return False
    
    return VisitorBlacklist.objects.filter(query).filter(
        Q(is_permanent=True) | Q(valid_until__gt=timezone.now().date())
    ).exists()
```

---

## Best Practices

1. **ID Verification**: Always verify visitor ID before issuing pass
2. **Photo Capture**: Capture visitor photo for security
3. **Host Notification**: Notify host when visitor arrives
4. **Time Tracking**: Track check-in and check-out times
5. **Blacklist Check**: Always check blacklist before registration

---
