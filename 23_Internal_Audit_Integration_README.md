# Internal Audit Module - Integration Guide

**What this module is:** The Internal Audit module will manage compliance audits, process reviews, audit scheduling, findings tracking, and corrective action monitoring.

**Why we need to build this:** Institutions need internal controls but currently audits are conducted manually with findings tracked in documents. This module will provide structured audit scheduling, digital findings management, and corrective action tracking.

**Why integration is needed:** Audits examine data from ALL modules - finance records, academic processes, inventory. Auditors and auditees are employees (ExtraInfo). Department audits link to DepartmentInfo. The module reads across the entire ERP for compliance checks.

**Key dependencies:** ExtraInfo, DepartmentInfo, ALL modules (for audit data)

---

## Module Overview
Manages internal audits, compliance checks, and audit reports.

## Core Dependencies
- `globals.ExtraInfo` - Auditor/Auditee
- `globals.DepartmentInfo` - Department audits
- All modules (for compliance data)

---

## Key Models

```python
from django.db import models
from applications.globals.models import ExtraInfo, DepartmentInfo

class AuditPlan(models.Model):
    """Annual/quarterly audit plans"""
    name = models.CharField(max_length=200)
    fiscal_year = models.CharField(max_length=10)
    
    # Scope
    departments = models.ManyToManyField(DepartmentInfo)
    modules = models.TextField(blank=True)  # JSON list
    
    # Timeline
    planned_start = models.DateField()
    planned_end = models.DateField()
    
    # Auditors
    lead_auditor = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='lead_audits')
    auditors = models.ManyToManyField(ExtraInfo, related_name='audit_assignments')
    
    status = models.CharField(max_length=20, default='PLANNED')
    
    created_at = models.DateTimeField(auto_now_add=True)

class AuditExecution(models.Model):
    """Individual audit executions"""
    STATUS = [
        ('SCHEDULED', 'Scheduled'),
        ('IN_PROGRESS', 'In Progress'),
        ('DRAFT_REPORT', 'Draft Report'),
        ('REVIEW', 'Under Review'),
        ('COMPLETED', 'Completed'),
    ]
    
    audit_plan = models.ForeignKey(AuditPlan, on_delete=models.CASCADE)
    department = models.ForeignKey(DepartmentInfo, on_delete=models.CASCADE)
    
    # Timeline
    scheduled_date = models.DateField()
    actual_date = models.DateField(null=True)
    completed_date = models.DateField(null=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='SCHEDULED')
    
    # Contact
    auditee_contact = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, related_name='audits_received')

class AuditFinding(models.Model):
    """Audit findings/observations"""
    SEVERITY = [
        ('LOW', 'Low'),
        ('MEDIUM', 'Medium'),
        ('HIGH', 'High'),
        ('CRITICAL', 'Critical'),
    ]
    
    FINDING_TYPE = [
        ('NON_COMPLIANCE', 'Non-Compliance'),
        ('OBSERVATION', 'Observation'),
        ('IMPROVEMENT', 'Improvement Opportunity'),
    ]
    
    audit = models.ForeignKey(AuditExecution, on_delete=models.CASCADE, related_name='findings')
    
    finding_type = models.CharField(max_length=20, choices=FINDING_TYPE)
    severity = models.CharField(max_length=20, choices=SEVERITY)
    
    title = models.CharField(max_length=300)
    description = models.TextField()
    evidence = models.TextField(blank=True)
    
    recommendation = models.TextField()
    
    # Response
    management_response = models.TextField(blank=True)
    corrective_action = models.TextField(blank=True)
    target_date = models.DateField(null=True)
    
    # Closure
    is_closed = models.BooleanField(default=False)
    closed_date = models.DateField(null=True)
    closure_remarks = models.TextField(blank=True)

class AuditReport(models.Model):
    """Final audit reports"""
    audit = models.OneToOneField(AuditExecution, on_delete=models.CASCADE, related_name='report')
    
    executive_summary = models.TextField()
    scope = models.TextField()
    methodology = models.TextField()
    
    overall_rating = models.CharField(max_length=50)  # Satisfactory, Needs Improvement, etc.
    
    report_date = models.DateField()
    report_document = models.FileField(upload_to='audit/reports/')
    
    reviewed_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    approved_at = models.DateTimeField(null=True)
```

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `AuditPlan` | Annual/quarterly audit plans |
| `AuditExecution` | Individual audit executions |
| `AuditFinding` | Audit findings/observations |
| `AuditReport` | Final audit reports |

---

## Integration with globals

### Get Auditor Details
```python
def get_auditor_details(auditor_id):
    """Get auditor details from globals"""
    from applications.globals.models import ExtraInfo
    
    auditor = ExtraInfo.objects.select_related('user', 'department').get(id=auditor_id)
    
    return {
        'id': auditor.id,
        'name': f"{auditor.user.first_name} {auditor.user.last_name}",
        'email': auditor.user.email,
        'department': auditor.department.name if auditor.department else None,
    }
```

### Get Department for Audit
```python
def get_department_info(department_id):
    """Get department info for audit"""
    from applications.globals.models import DepartmentInfo
    
    dept = DepartmentInfo.objects.get(id=department_id)
    return {
        'id': dept.id,
        'name': dept.name,
    }
```

### Get All Departments
```python
def get_all_departments():
    """Get all departments for audit scope"""
    from applications.globals.models import DepartmentInfo
    
    return DepartmentInfo.objects.all().values('id', 'name')
```

---

## Common Operations

### Create Audit Plan
```python
def create_audit_plan(name, fiscal_year, department_ids, planned_start, planned_end, lead_auditor_id, auditor_ids):
    """Create a new audit plan"""
    from applications.globals.models import ExtraInfo, DepartmentInfo
    
    lead_auditor = ExtraInfo.objects.get(id=lead_auditor_id)
    
    plan = AuditPlan.objects.create(
        name=name,
        fiscal_year=fiscal_year,
        planned_start=planned_start,
        planned_end=planned_end,
        lead_auditor=lead_auditor,
        status='PLANNED'
    )
    
    # Add departments
    departments = DepartmentInfo.objects.filter(id__in=department_ids)
    plan.departments.set(departments)
    
    # Add auditors
    auditors = ExtraInfo.objects.filter(id__in=auditor_ids)
    plan.auditors.set(auditors)
    
    return plan
```

### Schedule Audit Execution
```python
def schedule_audit(audit_plan_id, department_id, scheduled_date, auditee_contact_id):
    """Schedule an individual audit execution"""
    from applications.globals.models import ExtraInfo, DepartmentInfo
    
    plan = AuditPlan.objects.get(id=audit_plan_id)
    department = DepartmentInfo.objects.get(id=department_id)
    auditee = ExtraInfo.objects.get(id=auditee_contact_id) if auditee_contact_id else None
    
    return AuditExecution.objects.create(
        audit_plan=plan,
        department=department,
        scheduled_date=scheduled_date,
        auditee_contact=auditee,
        status='SCHEDULED'
    )
```

### Add Audit Finding
```python
def add_finding(audit_id, finding_type, severity, title, description, recommendation, evidence=''):
    """Add a finding to an audit"""
    audit = AuditExecution.objects.get(id=audit_id)
    
    return AuditFinding.objects.create(
        audit=audit,
        finding_type=finding_type,
        severity=severity,
        title=title,
        description=description,
        evidence=evidence,
        recommendation=recommendation
    )
```

### Close Finding
```python
def close_finding(finding_id, closure_remarks):
    """Close an audit finding"""
    from django.utils import timezone
    
    finding = AuditFinding.objects.get(id=finding_id)
    
    if not finding.corrective_action:
        raise ValueError("Corrective action must be documented before closing")
    
    finding.is_closed = True
    finding.closed_date = timezone.now().date()
    finding.closure_remarks = closure_remarks
    finding.save()
    
    return finding
```

### Generate Audit Report
```python
def generate_report(audit_id, executive_summary, scope, methodology, overall_rating, reviewer_id):
    """Generate audit report"""
    from django.utils import timezone
    from applications.globals.models import ExtraInfo
    
    audit = AuditExecution.objects.get(id=audit_id)
    reviewer = ExtraInfo.objects.get(id=reviewer_id)
    
    report = AuditReport.objects.create(
        audit=audit,
        executive_summary=executive_summary,
        scope=scope,
        methodology=methodology,
        overall_rating=overall_rating,
        report_date=timezone.now().date(),
        reviewed_by=reviewer
    )
    
    # Update audit status
    audit.status = 'DRAFT_REPORT'
    audit.save()
    
    return report
```

---

## Best Practices

1. **Planning**: Create annual audit plans covering all departments
2. **Documentation**: Document all findings with evidence
3. **Follow-up**: Track corrective actions to closure
4. **Independence**: Maintain auditor independence
5. **Reporting**: Generate comprehensive audit reports

---
