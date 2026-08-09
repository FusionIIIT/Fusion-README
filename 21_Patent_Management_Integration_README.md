# Patent Management Module - Integration Guide

**What this module is:** The Patent Management module will track intellectual property - invention disclosures, patent applications, filing status, inventor details, and technology transfer.

**Why we need to build this:** Research institutes generate IP but currently patent tracking is manual. Faculty disclose inventions on paper, filing status is tracked in spreadsheets. This module will provide digital invention disclosure, application tracking, and IP portfolio management.

**Why integration is needed:** Inventors are faculty (Faculty) or students (Student) identified by ExtraInfo. Patents are filed from departments (DepartmentInfo). The module needs identity and organizational data from globals to track who invented what.

**Key dependencies:** ExtraInfo, Faculty, Student, DepartmentInfo

---

## Module Overview
Manages intellectual property, patent applications, and technology transfer.

## Core Dependencies
- `globals.ExtraInfo` - Inventor details
- `globals.Faculty` - Faculty inventors
- `globals.DepartmentInfo` - Department reference

---

## Key Models

```python
from django.db import models
from applications.globals.models import ExtraInfo, Faculty, DepartmentInfo

class Patent(models.Model):
    """Patent applications"""
    STATUS = [
        ('DRAFT', 'Draft'),
        ('FILED', 'Filed'),
        ('PUBLISHED', 'Published'),
        ('EXAMINED', 'Under Examination'),
        ('GRANTED', 'Granted'),
        ('REJECTED', 'Rejected'),
        ('ABANDONED', 'Abandoned'),
    ]
    
    PATENT_TYPE = [
        ('UTILITY', 'Utility Patent'),
        ('DESIGN', 'Design Patent'),
        ('PLANT', 'Plant Patent'),
    ]
    
    title = models.CharField(max_length=500)
    patent_type = models.CharField(max_length=20, choices=PATENT_TYPE)
    
    # Inventors
    inventors = models.ManyToManyField(ExtraInfo, related_name='patents')
    lead_inventor = models.ForeignKey(ExtraInfo, on_delete=models.CASCADE, related_name='lead_patents')
    
    department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True)
    
    # Details
    abstract = models.TextField()
    description = models.TextField()
    claims = models.TextField()
    
    # Filing
    filing_date = models.DateField(null=True)
    filing_number = models.CharField(max_length=100, blank=True)
    
    # Patent office
    patent_office = models.CharField(max_length=100)  # Indian, US, EPO, etc.
    patent_number = models.CharField(max_length=100, blank=True)  # After grant
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='DRAFT')
    
    # Dates
    publication_date = models.DateField(null=True)
    grant_date = models.DateField(null=True)
    expiry_date = models.DateField(null=True)
    
    # Commercialization
    is_licensed = models.BooleanField(default=False)
    license_revenue = models.DecimalField(max_digits=15, decimal_places=2, default=0)

class PatentDocument(models.Model):
    """Patent related documents"""
    patent = models.ForeignKey(Patent, on_delete=models.CASCADE, related_name='documents')
    
    document_type = models.CharField(max_length=100)  # Application, Specification, etc.
    document = models.FileField(upload_to='patents/documents/')
    
    uploaded_at = models.DateTimeField(auto_now_add=True)
    uploaded_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)

class PatentExpense(models.Model):
    """Patent filing/maintenance expenses"""
    patent = models.ForeignKey(Patent, on_delete=models.CASCADE, related_name='expenses')
    
    expense_type = models.CharField(max_length=100)  # Filing fee, Attorney fee, etc.
    amount = models.DecimalField(max_digits=12, decimal_places=2)
    date = models.DateField()
    
    receipt = models.FileField(upload_to='patents/expenses/', blank=True)
    remarks = models.TextField(blank=True)
```

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `Patent` | Patent applications |
| `PatentDocument` | Patent related documents |
| `PatentExpense` | Patent filing/maintenance expenses |

---

## Integration with globals

### Get Inventor Details
```python
def get_inventor_details(extrainfo_id):
    """Get inventor details from globals"""
    from applications.globals.models import ExtraInfo, Faculty
    
    extrainfo = ExtraInfo.objects.select_related('user', 'department').get(id=extrainfo_id)
    
    inventor = {
        'id': extrainfo.id,
        'name': f"{extrainfo.user.first_name} {extrainfo.user.last_name}",
        'email': extrainfo.user.email,
        'department': extrainfo.department.name if extrainfo.department else None,
    }
    
    # Check if faculty
    faculty = Faculty.objects.filter(id=extrainfo).first()
    if faculty:
        inventor['designation'] = faculty.designation.name if faculty.designation else None
    
    return inventor
```

### Get Faculty Patents
```python
def get_faculty_patents(faculty_id):
    """Get all patents for a faculty member"""
    from applications.globals.models import ExtraInfo
    
    extrainfo = ExtraInfo.objects.get(id=faculty_id)
    
    # Patents as lead inventor
    lead_patents = Patent.objects.filter(lead_inventor=extrainfo)
    
    # Patents as co-inventor
    co_patents = Patent.objects.filter(inventors=extrainfo).exclude(lead_inventor=extrainfo)
    
    return {
        'lead_patents': lead_patents,
        'co_patents': co_patents,
        'total': lead_patents.count() + co_patents.count()
    }
```

### Get Department Patents
```python
def get_department_patents(department_id):
    """Get all patents from a department"""
    from applications.globals.models import DepartmentInfo
    
    department = DepartmentInfo.objects.get(id=department_id)
    
    return Patent.objects.filter(department=department).select_related(
        'lead_inventor', 'department'
    ).prefetch_related('inventors')
```

---

## Common Operations

### Create Patent Application
```python
def create_patent(title, patent_type, lead_inventor_id, inventors_ids, department_id, abstract, description, claims, patent_office):
    """Create a new patent application"""
    from applications.globals.models import ExtraInfo, DepartmentInfo
    
    lead_inventor = ExtraInfo.objects.get(id=lead_inventor_id)
    department = DepartmentInfo.objects.get(id=department_id) if department_id else None
    
    patent = Patent.objects.create(
        title=title,
        patent_type=patent_type,
        lead_inventor=lead_inventor,
        department=department,
        abstract=abstract,
        description=description,
        claims=claims,
        patent_office=patent_office,
        status='DRAFT'
    )
    
    # Add inventors
    inventors = ExtraInfo.objects.filter(id__in=inventors_ids)
    patent.inventors.set(inventors)
    
    return patent
```

### Update Patent Status
```python
def update_patent_status(patent_id, new_status, **kwargs):
    """Update patent status with relevant dates"""
    from django.utils import timezone
    
    patent = Patent.objects.get(id=patent_id)
    patent.status = new_status
    
    if new_status == 'FILED':
        patent.filing_date = kwargs.get('filing_date', timezone.now().date())
        patent.filing_number = kwargs.get('filing_number', '')
    elif new_status == 'PUBLISHED':
        patent.publication_date = kwargs.get('publication_date', timezone.now().date())
    elif new_status == 'GRANTED':
        patent.grant_date = kwargs.get('grant_date', timezone.now().date())
        patent.patent_number = kwargs.get('patent_number', '')
        # Calculate expiry (typically 20 years from filing)
        if patent.filing_date:
            patent.expiry_date = patent.filing_date.replace(year=patent.filing_date.year + 20)
    
    patent.save()
    return patent
```

### Record Patent Expense
```python
def add_patent_expense(patent_id, expense_type, amount, date, receipt=None, remarks=''):
    """Add expense record for a patent"""
    patent = Patent.objects.get(id=patent_id)
    
    return PatentExpense.objects.create(
        patent=patent,
        expense_type=expense_type,
        amount=amount,
        date=date,
        receipt=receipt,
        remarks=remarks
    )
```

---

## Best Practices

1. **Documentation**: Maintain complete records of all patent documents
2. **Inventor Consent**: Get signed consent from all inventors
3. **Prior Art Search**: Conduct thorough prior art search before filing
4. **Timely Filing**: File within statutory deadlines
5. **Expense Tracking**: Track all patent-related expenses

---
