# Backup & Archive System - Integration Guide

**What this module is:** The Backup & Archive module will manage data backups, archival of old records, retention policies, and disaster recovery procedures.

**Why we need to build this:** Data is critical but currently there's no systematic backup strategy. Graduated students' records need archival. Active data needs regular backups. This module will ensure data persistence, define retention policies, and enable disaster recovery.

**Why integration is needed:** Backups touch ALL tables. Archival policies may differ by data type (student records kept longer than logs). The module needs to understand schema structure across all production modules to backup and restore correctly.

**Key dependencies:** ALL tables (for backup/restore), system configuration

---

## Module Overview
Manages automated backups, data archival, retention policies, and disaster recovery.

## Core Dependencies
- All modules (for backup data)
- `globals.ExtraInfo` - User context
- `globals.DepartmentInfo` - Department-wise backups

---

## Key Models

```python
from django.db import models
from django.contrib.auth.models import User

class BackupConfiguration(models.Model):
    """Backup job configurations"""
    BACKUP_TYPE = [
        ('FULL', 'Full Backup'),
        ('INCREMENTAL', 'Incremental'),
        ('DIFFERENTIAL', 'Differential'),
    ]
    
    name = models.CharField(max_length=100)
    backup_type = models.CharField(max_length=20, choices=BACKUP_TYPE)
    
    # Schedule (cron format)
    schedule = models.CharField(max_length=100)
    
    # Target modules
    modules = models.TextField(blank=True)  # JSON list
    
    # Storage
    storage_path = models.CharField(max_length=500)
    retention_days = models.IntegerField(default=30)
    
    is_active = models.BooleanField(default=True)

class BackupRecord(models.Model):
    """Backup execution records"""
    STATUS = [
        ('RUNNING', 'Running'),
        ('SUCCESS', 'Success'),
        ('FAILED', 'Failed'),
    ]
    
    configuration = models.ForeignKey(BackupConfiguration, on_delete=models.CASCADE)
    
    started_at = models.DateTimeField()
    completed_at = models.DateTimeField(null=True)
    
    status = models.CharField(max_length=20, choices=STATUS)
    
    file_path = models.CharField(max_length=500, blank=True)
    file_size = models.BigIntegerField(null=True)  # bytes
    
    error_message = models.TextField(blank=True)

class ArchivePolicy(models.Model):
    """Data archival policies"""
    module = models.CharField(max_length=100)
    model_name = models.CharField(max_length=100)
    
    archive_after_days = models.IntegerField()  # Archive records older than X days
    delete_after_days = models.IntegerField(null=True)  # Delete archived after Y days
    
    is_active = models.BooleanField(default=True)

class ArchivedRecord(models.Model):
    """Archived data tracking"""
    module = models.CharField(max_length=100)
    model_name = models.CharField(max_length=100)
    original_id = models.CharField(max_length=100)
    
    data = models.TextField()  # JSON serialized data
    
    archived_at = models.DateTimeField(auto_now_add=True)
    archived_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
```

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `BackupConfiguration` | Backup job configurations |
| `BackupRecord` | Backup execution records |
| `ArchivePolicy` | Data archival policies |
| `ArchivedRecord` | Archived data tracking |

---

## Integration Points

### With All Modules
```python
def backup_module_data(module_name):
    """Backup data from any module"""
    from django.apps import apps
    
    # Get all models from module
    app_config = apps.get_app_config(module_name)
    models = app_config.get_models()
    
    backup_data = {}
    for model in models:
        backup_data[model.__name__] = list(model.objects.all().values())
    
    return backup_data
```

### With globals.ExtraInfo
```python
def get_backup_context(user):
    """Get user context for backup operations"""
    from applications.globals.models import ExtraInfo
    
    extrainfo = ExtraInfo.objects.get(user=user)
    return {
        'user_id': extrainfo.id,
        'department': extrainfo.department,
    }
```

---

## Common Operations

### Create Backup
```python
def create_backup(config_id, user):
    """Execute a backup based on configuration"""
    import json
    from datetime import datetime
    
    config = BackupConfiguration.objects.get(id=config_id)
    
    # Create record
    record = BackupRecord.objects.create(
        configuration=config,
        started_at=datetime.now(),
        status='RUNNING'
    )
    
    try:
        modules = json.loads(config.modules) if config.modules else []
        
        for module in modules:
            data = backup_module_data(module)
            # Save to storage_path
        
        record.status = 'SUCCESS'
        record.completed_at = datetime.now()
    except Exception as e:
        record.status = 'FAILED'
        record.error_message = str(e)
    
    record.save()
    return record
```

### Archive Old Records
```python
def archive_records(policy_id):
    """Archive records based on policy"""
    from django.apps import apps
    from django.utils import timezone
    from datetime import timedelta
    import json
    
    policy = ArchivePolicy.objects.get(id=policy_id)
    
    # Get model
    model = apps.get_model(policy.module, policy.model_name)
    
    # Find old records
    cutoff_date = timezone.now() - timedelta(days=policy.archive_after_days)
    old_records = model.objects.filter(created_at__lt=cutoff_date)
    
    for record in old_records:
        ArchivedRecord.objects.create(
            module=policy.module,
            model_name=policy.model_name,
            original_id=str(record.id),
            data=json.dumps(record.__dict__, default=str)
        )
        record.delete()
```

---

## Best Practices

1. **Schedule**: Run full backups weekly, incremental daily
2. **Retention**: Keep backups for at least 30 days
3. **Verification**: Regularly test backup restoration
4. **Encryption**: Encrypt sensitive backup data
5. **Offsite**: Store backups in multiple locations

---
