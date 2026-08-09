# System Security Module - Integration Guide

**What this module is:** The System Security module will manage authentication, authorization, access control lists, security policies, threat monitoring, and compliance.

**Why we need to build this:** ERP systems contain sensitive data - grades, finances, personal info. Currently security is basic. This module will enforce security policies, implement role-based access control, detect unauthorized access, and ensure compliance.

**Why integration is needed:** Security spans ALL modules. Access control checks user roles (from ExtraInfo). Audit logs track who accessed what data. The module monitors all API endpoints and database access across Fusion.

**Key dependencies:** Django User model, ExtraInfo, ALL modules (for access control)

---

## Module Overview
Manages security policies, access controls, threat monitoring, and compliance.

## Core Dependencies
- `django.contrib.auth.models.User` - User authentication
- `globals.ExtraInfo` - User context
- All modules (for access control)

---

## Key Models

```python
from django.db import models
from django.contrib.auth.models import User

class SecurityPolicy(models.Model):
    """Security policies"""
    name = models.CharField(max_length=200)
    description = models.TextField()
    
    # Password policy
    min_password_length = models.IntegerField(default=8)
    require_uppercase = models.BooleanField(default=True)
    require_numbers = models.BooleanField(default=True)
    require_special_chars = models.BooleanField(default=True)
    password_expiry_days = models.IntegerField(default=90)
    
    # Session policy
    session_timeout_minutes = models.IntegerField(default=30)
    max_concurrent_sessions = models.IntegerField(default=3)
    
    # Login policy
    max_login_attempts = models.IntegerField(default=5)
    lockout_duration_minutes = models.IntegerField(default=30)
    
    is_active = models.BooleanField(default=True)

class SecurityAlert(models.Model):
    """Security alerts"""
    SEVERITY = [
        ('LOW', 'Low'),
        ('MEDIUM', 'Medium'),
        ('HIGH', 'High'),
        ('CRITICAL', 'Critical'),
    ]
    
    ALERT_TYPE = [
        ('BRUTE_FORCE', 'Brute Force Attempt'),
        ('UNAUTHORIZED_ACCESS', 'Unauthorized Access'),
        ('DATA_BREACH', 'Potential Data Breach'),
        ('ANOMALY', 'Anomaly Detected'),
    ]
    
    alert_type = models.CharField(max_length=30, choices=ALERT_TYPE)
    severity = models.CharField(max_length=20, choices=SEVERITY)
    
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    ip_address = models.GenericIPAddressField(null=True)
    
    description = models.TextField()
    
    created_at = models.DateTimeField(auto_now_add=True)
    resolved = models.BooleanField(default=False)
    resolved_at = models.DateTimeField(null=True)

class IPBlocklist(models.Model):
    """Blocked IP addresses"""
    ip_address = models.GenericIPAddressField()
    reason = models.CharField(max_length=200)
    
    blocked_at = models.DateTimeField(auto_now_add=True)
    blocked_until = models.DateTimeField(null=True)  # Null = permanent
    
    blocked_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    is_active = models.BooleanField(default=True)

class TwoFactorAuth(models.Model):
    """2FA configuration per user"""
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    
    is_enabled = models.BooleanField(default=False)
    method = models.CharField(max_length=20)  # TOTP, SMS, Email
    
    secret_key = models.CharField(max_length=100, blank=True)  # For TOTP
    backup_codes = models.TextField(blank=True)  # JSON list
    
    enabled_at = models.DateTimeField(null=True)
```

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `SecurityPolicy` | Security policies configuration |
| `SecurityAlert` | Security alerts and incidents |
| `IPBlocklist` | Blocked IP addresses |
| `TwoFactorAuth` | Two-factor authentication settings |

---

## Integration Points

### With Django Auth
```python
def validate_password(password, policy=None):
    """Validate password against security policy"""
    if not policy:
        policy = SecurityPolicy.objects.filter(is_active=True).first()
    
    if not policy:
        return True, None
    
    errors = []
    
    if len(password) < policy.min_password_length:
        errors.append(f"Password must be at least {policy.min_password_length} characters")
    
    if policy.require_uppercase and not any(c.isupper() for c in password):
        errors.append("Password must contain uppercase letters")
    
    if policy.require_numbers and not any(c.isdigit() for c in password):
        errors.append("Password must contain numbers")
    
    if policy.require_special_chars and not any(c in '!@#$%^&*()' for c in password):
        errors.append("Password must contain special characters")
    
    return len(errors) == 0, errors
```

### With globals.ExtraInfo
```python
def get_user_security_context(user):
    """Get security context for user"""
    from applications.globals.models import ExtraInfo
    
    extrainfo = ExtraInfo.objects.get(user=user)
    tfa = TwoFactorAuth.objects.filter(user=user).first()
    
    return {
        'user_id': extrainfo.id,
        'user_type': extrainfo.user_type,
        'has_2fa': tfa.is_enabled if tfa else False,
    }
```

---

## Common Operations

### Check IP Blocklist
```python
def is_ip_blocked(ip_address):
    """Check if IP is blocked"""
    from django.utils import timezone
    
    blocked = IPBlocklist.objects.filter(
        ip_address=ip_address,
        is_active=True
    ).filter(
        models.Q(blocked_until__isnull=True) |
        models.Q(blocked_until__gt=timezone.now())
    ).exists()
    
    return blocked
```

### Log Security Alert
```python
def log_security_alert(alert_type, severity, user=None, ip=None, description=''):
    """Log a security alert"""
    return SecurityAlert.objects.create(
        alert_type=alert_type,
        severity=severity,
        user=user,
        ip_address=ip,
        description=description
    )
```

### Track Login Attempts
```python
def check_login_attempts(username, ip_address):
    """Check and track login attempts"""
    from django.core.cache import cache
    
    policy = SecurityPolicy.objects.filter(is_active=True).first()
    if not policy:
        return True
    
    cache_key = f"login_attempts_{username}_{ip_address}"
    attempts = cache.get(cache_key, 0)
    
    if attempts >= policy.max_login_attempts:
        # Block IP
        IPBlocklist.objects.create(
            ip_address=ip_address,
            reason=f"Too many failed login attempts for {username}",
            blocked_until=timezone.now() + timedelta(minutes=policy.lockout_duration_minutes)
        )
        return False
    
    return True

def record_failed_login(username, ip_address):
    """Record a failed login attempt"""
    from django.core.cache import cache
    
    cache_key = f"login_attempts_{username}_{ip_address}"
    attempts = cache.get(cache_key, 0)
    cache.set(cache_key, attempts + 1, timeout=1800)  # 30 min expiry
```

---

## Best Practices

1. **Password Policy**: Enforce strong passwords with regular rotation
2. **2FA**: Enable two-factor authentication for admin users
3. **IP Monitoring**: Monitor and block suspicious IP addresses
4. **Audit Logging**: Log all security-related events
5. **Session Management**: Implement session timeouts and concurrent session limits

---
