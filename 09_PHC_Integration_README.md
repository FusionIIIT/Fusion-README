# Primary Health Center (PHC) Module - Integration Guide

**What this module is:** The PHC module will manage the institute's health center - patient records, consultations, prescriptions, medical history, ambulance services, and health insurance claims for students and employees.

**Why we need to build this:** Students and staff need healthcare services on campus. Currently medical records are paper-based, making history tracking difficult. This module will digitize health center operations - appointments, prescriptions, medical history.

**Why integration is needed:** Patients are either students (from Student table) or employees (from ExtraInfo/Faculty/Staff). The PHC needs to identify patients, access their basic information (batch, department, contact details), and link medical records to the right person. All identity data comes from core modules.

**Key dependencies:** Student, ExtraInfo, Faculty, Staff

---

## Module Overview
The Primary Health Center module manages student and employee health records, medical consultations, prescriptions, ambulance services, health insurance, and wellness programs.

---

## Core Dependencies - Tables to Sync With

### 1. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in PHC |
|--------|------|-------------|--------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Patient (Student)** |
| `batch_id` | ForeignKey(Batch) | Batch | Student batch info |

---

### 2. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in PHC |
|--------|------|-------------|--------------|
| `id` | CharField(20) | User ID | **Patient ID** |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | **student/faculty/staff** |
| `sex` | CharField(2) | Gender | Medical history |
| `date_of_birth` | DateField | DOB | **Age calculation** |
| `address` | TextField | Address | Emergency contact |
| `phone_no` | BigIntegerField | Phone | **Emergency contact** |
| `department` | ForeignKey(DepartmentInfo) | Department | Patient department |

#### Table: `Faculty`
| Column | Type | Description | Usage in PHC |
|--------|------|-------------|--------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Patient (Faculty)** |

#### Table: `Staff`
| Column | Type | Description | Usage in PHC |
|--------|------|-------------|--------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Patient (Staff)** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in PHC |
|--------|------|-------------|--------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | Patient department |

---

## Data Models to Create in PHC Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, DepartmentInfo
from applications.academic_information.models import Student

# ==================== HEALTH PROFILE ====================

class BloodGroup(models.Model):
    """Blood group master"""
    group = models.CharField(max_length=10, unique=True)  # A+, A-, B+, B-, etc.

class HealthProfile(models.Model):
    """Patient health profile"""
    patient = models.OneToOneField(ExtraInfo, on_delete=models.CASCADE, related_name='health_profile')
    
    # Blood
    blood_group = models.ForeignKey(BloodGroup, on_delete=models.SET_NULL, null=True, blank=True)
    
    # Physical
    height = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)  # cm
    weight = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)  # kg
    
    # Medical history
    allergies = models.TextField(blank=True)
    chronic_conditions = models.TextField(blank=True)
    current_medications = models.TextField(blank=True)
    past_surgeries = models.TextField(blank=True)
    
    # Family history
    family_medical_history = models.TextField(blank=True)
    
    # Emergency
    emergency_contact_name = models.CharField(max_length=100, blank=True)
    emergency_contact_phone = models.CharField(max_length=15, blank=True)
    emergency_contact_relation = models.CharField(max_length=50, blank=True)
    
    # Insurance
    has_insurance = models.BooleanField(default=False)
    insurance_provider = models.CharField(max_length=100, blank=True)
    insurance_policy_number = models.CharField(max_length=50, blank=True)
    insurance_valid_until = models.DateField(null=True, blank=True)
    
    # Last updated
    updated_at = models.DateTimeField(auto_now=True)

# ==================== DOCTOR/STAFF ====================

class MedicalStaff(models.Model):
    """Medical staff at PHC"""
    STAFF_TYPE = [
        ('DOCTOR', 'Doctor'),
        ('NURSE', 'Nurse'),
        ('PHARMACIST', 'Pharmacist'),
        ('LAB_TECH', 'Lab Technician'),
        ('OTHER', 'Other'),
    ]
    
    user = models.OneToOneField(ExtraInfo, on_delete=models.CASCADE, related_name='medical_staff')
    staff_type = models.CharField(max_length=20, choices=STAFF_TYPE)
    specialization = models.CharField(max_length=100, blank=True)
    registration_number = models.CharField(max_length=50, blank=True)  # Medical council
    is_active = models.BooleanField(default=True)
    available_days = models.CharField(max_length=100, blank=True)  # JSON: ["Mon", "Tue"]
    duty_hours_start = models.TimeField(null=True, blank=True)
    duty_hours_end = models.TimeField(null=True, blank=True)

# ==================== APPOINTMENTS ====================

class Appointment(models.Model):
    """Patient appointments"""
    STATUS = [
        ('SCHEDULED', 'Scheduled'),
        ('CHECKED_IN', 'Checked In'),
        ('IN_PROGRESS', 'In Progress'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
        ('NO_SHOW', 'No Show'),
    ]
    
    APPOINTMENT_TYPE = [
        ('OPD', 'OPD'),
        ('EMERGENCY', 'Emergency'),
        ('FOLLOW_UP', 'Follow Up'),
        ('VACCINATION', 'Vaccination'),
        ('LAB_TEST', 'Lab Test'),
        ('WELLNESS_CHECK', 'Wellness Check'),
    ]
    
    patient = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='phc_appointments')
    doctor = models.ForeignKey(MedicalStaff, on_delete=models.SET_NULL, null=True, blank=True, related_name='appointments')
    
    appointment_type = models.CharField(max_length=20, choices=APPOINTMENT_TYPE, default='OPD')
    appointment_date = models.DateField()
    appointment_time = models.TimeField()
    
    chief_complaint = models.TextField(blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='SCHEDULED')
    
    created_at = models.DateTimeField(auto_now_add=True)
    check_in_time = models.DateTimeField(null=True, blank=True)
    consultation_start = models.DateTimeField(null=True, blank=True)
    consultation_end = models.DateTimeField(null=True, blank=True)

# ==================== CONSULTATION ====================

class Consultation(models.Model):
    """Medical consultation records"""
    appointment = models.OneToOneField(Appointment, on_delete=models.CASCADE, related_name='consultation')
    patient = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='consultations')
    doctor = models.ForeignKey(MedicalStaff, on_delete=models.SET_NULL, null=True)
    
    consultation_date = models.DateTimeField(auto_now_add=True)
    
    # Vitals
    blood_pressure_systolic = models.IntegerField(null=True, blank=True)
    blood_pressure_diastolic = models.IntegerField(null=True, blank=True)
    pulse_rate = models.IntegerField(null=True, blank=True)
    temperature = models.DecimalField(max_digits=4, decimal_places=1, null=True, blank=True)
    oxygen_saturation = models.IntegerField(null=True, blank=True)
    weight = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)
    
    # Clinical notes
    chief_complaint = models.TextField()
    history_of_present_illness = models.TextField(blank=True)
    examination_findings = models.TextField(blank=True)
    provisional_diagnosis = models.TextField(blank=True)
    final_diagnosis = models.TextField(blank=True)
    
    # ICD codes
    diagnosis_codes = models.TextField(blank=True)  # JSON list of ICD-10 codes
    
    # Recommendations
    treatment_plan = models.TextField(blank=True)
    advice = models.TextField(blank=True)
    follow_up_date = models.DateField(null=True, blank=True)
    
    # Referral
    referred_to_hospital = models.BooleanField(default=False)
    referral_details = models.TextField(blank=True)

# ==================== PRESCRIPTIONS ====================

class Medicine(models.Model):
    """Medicine master"""
    MEDICINE_TYPE = [
        ('TABLET', 'Tablet'),
        ('CAPSULE', 'Capsule'),
        ('SYRUP', 'Syrup'),
        ('INJECTION', 'Injection'),
        ('CREAM', 'Cream'),
        ('DROPS', 'Drops'),
        ('INHALER', 'Inhaler'),
        ('OTHER', 'Other'),
    ]
    
    name = models.CharField(max_length=200)
    generic_name = models.CharField(max_length=200, blank=True)
    medicine_type = models.CharField(max_length=20, choices=MEDICINE_TYPE)
    strength = models.CharField(max_length=50, blank=True)  # e.g., "500mg"
    manufacturer = models.CharField(max_length=200, blank=True)
    is_available = models.BooleanField(default=True)
    
    class Meta:
        ordering = ['name']

class Prescription(models.Model):
    """Prescriptions"""
    consultation = models.ForeignKey(Consultation, on_delete=models.CASCADE, related_name='prescriptions')
    medicine = models.ForeignKey(Medicine, on_delete=models.PROTECT, null=True, blank=True)
    medicine_name = models.CharField(max_length=200)  # Allow manual entry
    
    dosage = models.CharField(max_length=100)  # e.g., "1-0-1"
    frequency = models.CharField(max_length=100)  # e.g., "Twice daily"
    duration = models.CharField(max_length=50)  # e.g., "5 days"
    quantity = models.IntegerField(default=0)
    
    instructions = models.TextField(blank=True)  # Before/after food, etc.
    
    # Dispensing
    dispensed = models.BooleanField(default=False)
    dispensed_quantity = models.IntegerField(default=0)
    dispensed_at = models.DateTimeField(null=True, blank=True)
    dispensed_by = models.ForeignKey(MedicalStaff, on_delete=models.SET_NULL, null=True, blank=True)

# ==================== LAB TESTS ====================

class LabTest(models.Model):
    """Lab test master"""
    name = models.CharField(max_length=200)
    code = models.CharField(max_length=20, unique=True)
    description = models.TextField(blank=True)
    sample_type = models.CharField(max_length=50, blank=True)  # Blood, Urine, etc.
    normal_range = models.CharField(max_length=100, blank=True)
    turnaround_time = models.CharField(max_length=50, blank=True)  # e.g., "24 hours"
    is_available = models.BooleanField(default=True)

class LabOrder(models.Model):
    """Lab test orders"""
    STATUS = [
        ('ORDERED', 'Ordered'),
        ('SAMPLE_COLLECTED', 'Sample Collected'),
        ('PROCESSING', 'Processing'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    patient = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='lab_orders')
    consultation = models.ForeignKey(Consultation, on_delete=models.SET_NULL, null=True, blank=True)
    ordered_by = models.ForeignKey(MedicalStaff, on_delete=models.SET_NULL, null=True, related_name='lab_orders')
    
    order_date = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=20, choices=STATUS, default='ORDERED')
    
    clinical_notes = models.TextField(blank=True)
    urgent = models.BooleanField(default=False)

class LabOrderItem(models.Model):
    """Individual tests in an order"""
    lab_order = models.ForeignKey(LabOrder, on_delete=models.CASCADE, related_name='items')
    test = models.ForeignKey(LabTest, on_delete=models.PROTECT)
    
    result_value = models.CharField(max_length=200, blank=True)
    result_unit = models.CharField(max_length=50, blank=True)
    result_status = models.CharField(max_length=20, blank=True)  # Normal, High, Low
    result_date = models.DateTimeField(null=True, blank=True)
    result_by = models.ForeignKey(MedicalStaff, on_delete=models.SET_NULL, null=True, blank=True)
    
    remarks = models.TextField(blank=True)

# ==================== AMBULANCE ====================

class AmbulanceRequest(models.Model):
    """Ambulance service requests"""
    STATUS = [
        ('REQUESTED', 'Requested'),
        ('DISPATCHED', 'Dispatched'),
        ('ARRIVED', 'Arrived'),
        ('TRANSPORTING', 'Transporting'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    EMERGENCY_LEVEL = [
        ('CRITICAL', 'Critical'),
        ('URGENT', 'Urgent'),
        ('ROUTINE', 'Routine'),
    ]
    
    patient = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, null=True, blank=True, related_name='ambulance_requests')
    patient_name = models.CharField(max_length=100, blank=True)  # For unknown patients
    
    requested_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='ambulance_requests_made')
    
    emergency_level = models.CharField(max_length=20, choices=EMERGENCY_LEVEL, default='URGENT')
    pickup_location = models.TextField()
    destination = models.TextField()
    
    reason = models.TextField()
    
    status = models.CharField(max_length=20, choices=STATUS, default='REQUESTED')
    
    # Timeline
    requested_at = models.DateTimeField(auto_now_add=True)
    dispatched_at = models.DateTimeField(null=True, blank=True)
    arrived_at = models.DateTimeField(null=True, blank=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    
    # Driver
    driver_name = models.CharField(max_length=100, blank=True)
    driver_phone = models.CharField(max_length=15, blank=True)
    vehicle_number = models.CharField(max_length=20, blank=True)

# ==================== MEDICINE INVENTORY ====================

class MedicineStock(models.Model):
    """PHC medicine inventory"""
    medicine = models.ForeignKey(Medicine, on_delete=models.CASCADE)
    batch_number = models.CharField(max_length=50)
    
    quantity = models.IntegerField(default=0)
    received_date = models.DateField()
    expiry_date = models.DateField()
    
    unit_price = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    class Meta:
        unique_together = ['medicine', 'batch_number']

# ==================== MEDICAL CERTIFICATES ====================

class MedicalCertificate(models.Model):
    """Medical/fitness certificates"""
    CERTIFICATE_TYPE = [
        ('SICK_LEAVE', 'Sick Leave'),
        ('FITNESS', 'Fitness'),
        ('MEDICAL_EXCUSE', 'Medical Excuse'),
        ('DISABILITY', 'Disability'),
        ('OTHER', 'Other'),
    ]
    
    patient = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='medical_certificates')
    consultation = models.ForeignKey(Consultation, on_delete=models.SET_NULL, null=True, blank=True)
    
    certificate_type = models.CharField(max_length=20, choices=CERTIFICATE_TYPE)
    issue_date = models.DateField()
    
    # For sick leave
    from_date = models.DateField(null=True, blank=True)
    to_date = models.DateField(null=True, blank=True)
    
    diagnosis = models.TextField(blank=True)
    recommendations = models.TextField(blank=True)
    
    issued_by = models.ForeignKey(MedicalStaff, on_delete=models.SET_NULL, null=True)
    certificate_number = models.CharField(max_length=50, unique=True)
    
    document = models.FileField(upload_to='phc/certificates/', blank=True)

# ==================== VACCINATION ====================

class VaccinationDrive(models.Model):
    """Vaccination campaigns"""
    name = models.CharField(max_length=200)
    vaccine_name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    start_date = models.DateField()
    end_date = models.DateField()
    target_group = models.CharField(max_length=100, blank=True)  # Students, Faculty, All
    is_active = models.BooleanField(default=True)

class VaccinationRecord(models.Model):
    """Individual vaccination records"""
    patient = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='vaccinations')
    drive = models.ForeignKey(VaccinationDrive, on_delete=models.SET_NULL, null=True, blank=True)
    
    vaccine_name = models.CharField(max_length=200)
    dose_number = models.IntegerField(default=1)
    
    vaccination_date = models.DateField()
    batch_number = models.CharField(max_length=50, blank=True)
    
    administered_by = models.ForeignKey(MedicalStaff, on_delete=models.SET_NULL, null=True)
    
    next_dose_date = models.DateField(null=True, blank=True)
    remarks = models.TextField(blank=True)
```

---

## Integration Functions

### Get Patient Info
```python
def get_patient_info(user):
    """Get complete patient information"""
    from applications.globals.models import ExtraInfo
    from applications.academic_information.models import Student
    
    extrainfo = ExtraInfo.objects.get(user=user)
    
    patient_data = {
        'id': extrainfo.id,
        'name': f"{user.first_name} {user.last_name}",
        'user_type': extrainfo.user_type,
        'sex': extrainfo.sex,
        'dob': extrainfo.date_of_birth,
        'phone': extrainfo.phone_no,
        'department': extrainfo.department.name if extrainfo.department else None,
    }
    
    # If student, add batch info
    if extrainfo.user_type == 'student':
        try:
            student = Student.objects.get(id=extrainfo)
            patient_data['batch'] = str(student.batch_id) if student.batch_id else None
        except Student.DoesNotExist:
            pass
    
    return patient_data
```

### Calculate Patient Age
```python
def calculate_age(extrainfo):
    """Calculate patient age from DOB"""
    from datetime import date
    
    if extrainfo.date_of_birth:
        today = date.today()
        dob = extrainfo.date_of_birth
        age = today.year - dob.year - ((today.month, today.day) < (dob.month, dob.day))
        return age
    return None
```

### Check Insurance Status
```python
def check_insurance_status(patient_extrainfo):
    """Check if patient has valid insurance"""
    from datetime import date
    
    try:
        profile = patient_extrainfo.health_profile
        if profile.has_insurance and profile.insurance_valid_until:
            return profile.insurance_valid_until >= date.today()
    except:
        pass
    return False
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/phc/api/appointments/` | GET, POST | Appointments |
| `/phc/api/consultations/` | GET, POST | Consultations |
| `/phc/api/prescriptions/` | GET | Prescriptions |
| `/phc/api/lab-orders/` | GET, POST | Lab orders |
| `/phc/api/ambulance/` | GET, POST | Ambulance requests |
| `/phc/api/certificates/` | GET, POST | Medical certificates |
| `/phc/api/health-profile/` | GET, PUT | Health profile |

---

## Important Sync Points

1. **Patient Base**: Use `ExtraInfo` as patient identifier.
2. **Student/Faculty/Staff**: Check `user_type` to determine patient category.
3. **Demographics**: Get age from `date_of_birth`, gender from `sex`.
4. **Contact**: Emergency contact from `phone_no` or `HealthProfile`.
5. **Leave Integration**: Medical certificates link to Leave module.

---

