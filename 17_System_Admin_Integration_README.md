# System Administration Module - Integration Guide

**What this module is:** The System Administration module will manage user accounts, role assignments, permissions, system configurations, access controls, audit logs, and overall system settings.

**Why we need to build this:** Fusion has many modules and user types but currently user management is scattered. This module will provide centralized account creation, role assignment (student/faculty/admin), permission management, and system configuration.

**Why integration is needed:** System Admin controls ALL other modules. It assigns roles based on user type (Student/Faculty/Staff from globals). It configures module-specific settings. It needs access to all identity tables to manage user accounts and permissions.

**Key dependencies:** ExtraInfo, Student, Faculty, Staff, all module tables (for permissions)

---

## Module Overview
The System Administration module manages user accounts, role assignments, system configurations, access controls, audit logs, and overall system settings across all Fusion modules.

---

## Core Dependencies - Tables to Sync With

### 1. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in Admin |
|--------|------|-------------|----------------|
| `id` | CharField(20) | User ID | **User identifier** |
| `user` | OneToOneField(User) | User | **Django User** |
| `user_type` | CharField(20) | Type | **Role assignment** |
| `user_status` | CharField(50) | Status | **Active/Inactive** |
| `department` | ForeignKey(DepartmentInfo) | Department | **Access scope** |

#### Table: `Designation`
| Column | Type | Description | Usage in Admin |
|--------|------|-------------|----------------|
| `id` | AutoField | Primary Key | Designation reference |
| `name` | CharField(50) | Designation | **Role definition** |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in Admin |
|--------|------|-------------|----------------|
| `user` | ForeignKey(User) | User | **User-Role mapping** |
| `designation` | ForeignKey(Designation) | Designation | **Assigned role** |
| `held_at` | DateTimeField | Held from | Assignment date |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in Admin |
|--------|------|-------------|----------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | **Access scope** |

---

## Data Models to Create in System Admin Module

```python
from django.db import models
from django.contrib.auth.models import User, Permission
from applications.globals.models import ExtraInfo, Designation, DepartmentInfo

# ==================== ROLE & PERMISSION ====================

class Role(models.Model):
    """Custom roles beyond Django groups"""
    name = models.CharField(max_length=100, unique=True)
    description = models.TextField(blank=True)
    
    # Link to designation
    designation = models.ForeignKey(Designation, on_delete=models.SET_NULL, null=True, blank=True)
    
    # Module access
    allowed_modules = models.TextField(blank=True)  # JSON list
    
    # Permissions
    permissions = models.ManyToManyField(Permission, blank=True)
    
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

class UserRole(models.Model):
    """User-Role assignments"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='custom_roles')
    role = models.ForeignKey(Role, on_delete=models.CASCADE)
    
    # Scope
    department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    assigned_at = models.DateTimeField(auto_now_add=True)
    assigned_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True, related_name='role_assignments_made')
    
    valid_from = models.DateField(null=True, blank=True)
    valid_until = models.DateField(null=True, blank=True)
    
    is_active = models.BooleanField(default=True)
    
    class Meta:
        unique_together = ['user', 'role', 'department']

# ==================== MODULE ACCESS ====================

class ModulePermission(models.Model):
    """Module-level permissions"""
    PERMISSION_TYPE = [
        ('VIEW', 'View'),
        ('CREATE', 'Create'),
        ('EDIT', 'Edit'),
        ('DELETE', 'Delete'),
        ('APPROVE', 'Approve'),
        ('ADMIN', 'Admin'),
    ]
    
    module_name = models.CharField(max_length=100)
    permission_type = models.CharField(max_length=20, choices=PERMISSION_TYPE)
    
    description = models.TextField(blank=True)
    
    class Meta:
        unique_together = ['module_name', 'permission_type']

class RoleModulePermission(models.Model):
    """Role-Module permission mapping"""
    role = models.ForeignKey(Role, on_delete=models.CASCADE, related_name='module_permissions')
    module_permission = models.ForeignKey(ModulePermission, on_delete=models.CASCADE)
    
    class Meta:
        unique_together = ['role', 'module_permission']

# ==================== USER MANAGEMENT ====================

class UserAccountRequest(models.Model):
    """New account requests"""
    STATUS = [
        ('PENDING', 'Pending'),
        ('APPROVED', 'Approved'),
        ('REJECTED', 'Rejected'),
    ]
    
    USER_TYPE = [
        ('student', 'Student'),
        ('faculty', 'Faculty'),
        ('staff', 'Staff'),
    ]
    
    requested_username = models.CharField(max_length=150)
    requested_email = models.EmailField()
    requested_user_type = models.CharField(max_length=20, choices=USER_TYPE)
    
    first_name = models.CharField(max_length=150)
    last_name = models.CharField(max_length=150)
    
    department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True)
    
    reason = models.TextField()
    supporting_documents = models.FileField(upload_to='admin/account_requests/', blank=True)
    
    status = models.CharField(max_length=20, choices=STATUS, default='PENDING')
    
    requested_at = models.DateTimeField(auto_now_add=True)
    processed_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True, blank=True)
    processed_at = models.DateTimeField(null=True, blank=True)
    remarks = models.TextField(blank=True)

class UserAccountHistory(models.Model):
    """User account change history"""
    ACTION_TYPE = [
        ('CREATED', 'Created'),
        ('UPDATED', 'Updated'),
        ('ACTIVATED', 'Activated'),
        ('DEACTIVATED', 'Deactivated'),
        ('PASSWORD_RESET', 'Password Reset'),
        ('ROLE_CHANGED', 'Role Changed'),
        ('DELETED', 'Deleted'),
    ]
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='account_history')
    action = models.CharField(max_length=20, choices=ACTION_TYPE)
    
    details = models.TextField(blank=True)  # JSON details
    
    performed_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True, related_name='admin_actions')
    performed_at = models.DateTimeField(auto_now_add=True)
    
    ip_address = models.GenericIPAddressField(null=True, blank=True)

# ==================== AUDIT LOGGING ====================

class AuditLog(models.Model):
    """System-wide audit log"""
    ACTION_TYPE = [
        ('LOGIN', 'Login'),
        ('LOGOUT', 'Logout'),
        ('CREATE', 'Create'),
        ('READ', 'Read'),
        ('UPDATE', 'Update'),
        ('DELETE', 'Delete'),
        ('EXPORT', 'Export'),
        ('IMPORT', 'Import'),
        ('APPROVE', 'Approve'),
        ('REJECT', 'Reject'),
    ]
    
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    action = models.CharField(max_length=20, choices=ACTION_TYPE)
    
    # What was affected
    module = models.CharField(max_length=100)
    model_name = models.CharField(max_length=100, blank=True)
    object_id = models.CharField(max_length=100, blank=True)
    
    # Details
    description = models.TextField()
    old_values = models.TextField(blank=True)  # JSON
    new_values = models.TextField(blank=True)  # JSON
    
    # Context
    ip_address = models.GenericIPAddressField(null=True, blank=True)
    user_agent = models.TextField(blank=True)
    
    timestamp = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['-timestamp']
        indexes = [
            models.Index(fields=['user', 'timestamp']),
            models.Index(fields=['module', 'timestamp']),
        ]

class LoginHistory(models.Model):
    """User login history"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='login_history')
    
    login_time = models.DateTimeField()
    logout_time = models.DateTimeField(null=True, blank=True)
    
    ip_address = models.GenericIPAddressField(null=True, blank=True)
    user_agent = models.TextField(blank=True)
    device_type = models.CharField(max_length=50, blank=True)
    
    # Session
    session_key = models.CharField(max_length=100, blank=True)
    
    successful = models.BooleanField(default=True)
    failure_reason = models.CharField(max_length=200, blank=True)
    
    class Meta:
        ordering = ['-login_time']

# ==================== SYSTEM CONFIGURATION ====================

class SystemConfiguration(models.Model):
    """System-wide configurations"""
    CONFIG_TYPE = [
        ('STRING', 'String'),
        ('INTEGER', 'Integer'),
        ('BOOLEAN', 'Boolean'),
        ('JSON', 'JSON'),
    ]
    
    key = models.CharField(max_length=100, unique=True)
    value = models.TextField()
    config_type = models.CharField(max_length=20, choices=CONFIG_TYPE, default='STRING')
    
    description = models.TextField(blank=True)
    module = models.CharField(max_length=100, blank=True)  # Which module uses this
    
    is_editable = models.BooleanField(default=True)
    
    updated_at = models.DateTimeField(auto_now=True)
    updated_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)

class ModuleConfiguration(models.Model):
    """Per-module configurations"""
    module_name = models.CharField(max_length=100)
    
    is_enabled = models.BooleanField(default=True)
    maintenance_mode = models.BooleanField(default=False)
    maintenance_message = models.TextField(blank=True)
    
    # Access control
    allowed_user_types = models.TextField(blank=True)  # JSON list
    
    # Customization
    custom_settings = models.TextField(blank=True)  # JSON
    
    updated_at = models.DateTimeField(auto_now=True)

# ==================== NOTIFICATION SETTINGS ====================

class NotificationTemplate(models.Model):
    """Email/SMS notification templates"""
    TEMPLATE_TYPE = [
        ('EMAIL', 'Email'),
        ('SMS', 'SMS'),
        ('PUSH', 'Push Notification'),
    ]
    
    name = models.CharField(max_length=100, unique=True)
    template_type = models.CharField(max_length=20, choices=TEMPLATE_TYPE)
    module = models.CharField(max_length=100)
    
    subject = models.CharField(max_length=300, blank=True)
    body = models.TextField()
    
    # Variables used
    variables = models.TextField(blank=True)  # JSON list of variable names
    
    is_active = models.BooleanField(default=True)

class UserNotificationPreference(models.Model):
    """User notification preferences"""
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='notification_preferences')
    
    email_notifications = models.BooleanField(default=True)
    sms_notifications = models.BooleanField(default=False)
    push_notifications = models.BooleanField(default=True)
    
    # Module-specific
    module_preferences = models.TextField(blank=True)  # JSON

# ==================== SCHEDULED TASKS ====================

class ScheduledTask(models.Model):
    """System scheduled tasks"""
    STATUS = [
        ('ACTIVE', 'Active'),
        ('PAUSED', 'Paused'),
        ('DISABLED', 'Disabled'),
    ]
    
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True)
    
    task_function = models.CharField(max_length=200)  # Python function path
    schedule = models.CharField(max_length=100)  # Cron expression
    
    status = models.CharField(max_length=20, choices=STATUS, default='ACTIVE')
    
    last_run = models.DateTimeField(null=True, blank=True)
    next_run = models.DateTimeField(null=True, blank=True)
    
    last_status = models.CharField(max_length=50, blank=True)
    last_error = models.TextField(blank=True)

class TaskExecutionLog(models.Model):
    """Task execution history"""
    task = models.ForeignKey(ScheduledTask, on_delete=models.CASCADE, related_name='execution_logs')
    
    started_at = models.DateTimeField()
    ended_at = models.DateTimeField(null=True, blank=True)
    
    status = models.CharField(max_length=50)
    output = models.TextField(blank=True)
    error = models.TextField(blank=True)
    
    class Meta:
        ordering = ['-started_at']

# ==================== API MANAGEMENT ====================

class APIKey(models.Model):
    """API keys for external integrations"""
    name = models.CharField(max_length=100)
    key = models.CharField(max_length=100, unique=True)
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='api_keys')
    
    # Permissions
    allowed_modules = models.TextField(blank=True)  # JSON list
    rate_limit = models.IntegerField(default=1000)  # Requests per hour
    
    is_active = models.BooleanField(default=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    expires_at = models.DateTimeField(null=True, blank=True)
    last_used = models.DateTimeField(null=True, blank=True)

class APIAccessLog(models.Model):
    """API access log"""
    api_key = models.ForeignKey(APIKey, on_delete=models.SET_NULL, null=True)
    
    endpoint = models.CharField(max_length=500)
    method = models.CharField(max_length=10)
    
    request_data = models.TextField(blank=True)
    response_code = models.IntegerField()
    
    ip_address = models.GenericIPAddressField(null=True, blank=True)
    timestamp = models.DateTimeField(auto_now_add=True)
```

---

## Integration Functions

### Check User Permission
```python
def check_module_permission(user, module_name, permission_type):
    """Check if user has permission for a module"""
    # Check custom roles first
    user_roles = UserRole.objects.filter(
        user=user,
        is_active=True
    ).select_related('role')
    
    for user_role in user_roles:
        if RoleModulePermission.objects.filter(
            role=user_role.role,
            module_permission__module_name=module_name,
            module_permission__permission_type=permission_type
        ).exists():
            return True
    
    # Check HoldsDesignation
    from applications.globals.models import HoldsDesignation
    designations = HoldsDesignation.objects.filter(user=user)
    
    for hd in designations:
        roles = Role.objects.filter(designation=hd.designation, is_active=True)
        for role in roles:
            if RoleModulePermission.objects.filter(
                role=role,
                module_permission__module_name=module_name,
                module_permission__permission_type=permission_type
            ).exists():
                return True
    
    return False
```

### Log Audit Action
```python
def log_audit_action(user, action, module, description, model_name=None, object_id=None, old_values=None, new_values=None, request=None):
    """Create audit log entry"""
    import json
    
    log = AuditLog.objects.create(
        user=user,
        action=action,
        module=module,
        model_name=model_name or '',
        object_id=str(object_id) if object_id else '',
        description=description,
        old_values=json.dumps(old_values) if old_values else '',
        new_values=json.dumps(new_values) if new_values else '',
        ip_address=request.META.get('REMOTE_ADDR') if request else None,
        user_agent=request.META.get('HTTP_USER_AGENT', '')[:500] if request else '',
    )
    return log
```

### Get User's Effective Roles
```python
def get_user_roles(user):
    """Get all roles for a user"""
    from applications.globals.models import HoldsDesignation
    
    roles = []
    
    # Custom roles
    user_roles = UserRole.objects.filter(
        user=user,
        is_active=True
    ).select_related('role')
    
    for ur in user_roles:
        roles.append({
            'type': 'custom',
            'role': ur.role,
            'department': ur.department,
        })
    
    # From designations
    designations = HoldsDesignation.objects.filter(
        user=user
    ).select_related('designation')
    
    for hd in designations:
        designation_roles = Role.objects.filter(
            designation=hd.designation,
            is_active=True
        )
        for role in designation_roles:
            roles.append({
                'type': 'designation',
                'role': role,
                'designation': hd.designation,
            })
    
    return roles
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/admin/api/users/` | GET, POST | User management |
| `/admin/api/users/<id>/roles/` | GET, POST | User roles |
| `/admin/api/roles/` | GET, POST | Role management |
| `/admin/api/audit-logs/` | GET | Audit logs |
| `/admin/api/config/` | GET, PUT | System configuration |
| `/admin/api/modules/` | GET, PUT | Module settings |

---

## Important Sync Points

1. **User Base**: Use Django `User` model with `ExtraInfo` extension.
2. **Designations**: Sync roles with `HoldsDesignation` from globals.
3. **Departments**: Use `DepartmentInfo` for access scoping.
4. **User Type**: Reference `ExtraInfo.user_type` for categorization.

---

