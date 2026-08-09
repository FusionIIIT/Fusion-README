# Dashboards Module - Integration Guide

**What this module is:** The Dashboards module will provide role-based summary views - students see their courses and grades, faculty see their advisees and classes, admins see analytics and metrics.

**Why we need to build this:** Users need a personalized landing page. A student wants to see their courses, dues, notifications at a glance. A faculty wants their classes and pending approvals. Currently there's no unified dashboard. This module will aggregate data from all modules into role-based views.

**Why integration is needed:** Dashboards READ from ALL production modules. Student dashboards need course_registration, Student_grades, Dues. Faculty dashboards need thesis supervision, department info. It's the ultimate consumer of all production tables.

**Key dependencies:** ALL production tables (Student, course_registration, Student_grades, Dues, ExtraInfo, etc.)

---

## Module Overview
The Dashboards module provides role-based dashboards aggregating data from all Fusion modules, offering real-time analytics, key metrics, notifications, and quick actions for different user types (students, faculty, staff, administrators).

---

## Core Dependencies - Tables to Sync With

This module reads data from ALL other modules. The primary dependencies are:

### 1. From `globals` Module
- `ExtraInfo` - User information and type
- `Faculty` - Faculty details
- `Staff` - Staff details
- `DepartmentInfo` - Department information
- `HoldsDesignation` - Role/designation mappings

### 2. From `academic_information` Module
- `Student` - Student details
- `Student_grades` - Academic performance (from online_cms)

### 3. From `programme_curriculum` Module
- `Course` - Course details
- `CourseInstructor` - Teaching assignments
- `Batch` - Batch information

### 4. From `academic_procedures` Module
- `course_registration` - Registration data
- `Dues` - Pending dues
- `FeePayments` - Fee status

### 5. From Other Modules
- `online_cms` - Course materials, assignments
- Hostel, Mess, Placement, etc.

---

## Data Models to Create in Dashboards Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, DepartmentInfo

# ==================== DASHBOARD CONFIGURATION ====================

class DashboardWidget(models.Model):
    """Available dashboard widgets"""
    WIDGET_TYPE = [
        ('COUNTER', 'Counter'),
        ('CHART', 'Chart'),
        ('TABLE', 'Table'),
        ('LIST', 'List'),
        ('CALENDAR', 'Calendar'),
        ('NOTIFICATION', 'Notification'),
        ('QUICK_ACTION', 'Quick Action'),
    ]
    
    USER_TYPE = [
        ('student', 'Student'),
        ('faculty', 'Faculty'),
        ('staff', 'Staff'),
        ('admin', 'Admin'),
        ('all', 'All'),
    ]
    
    name = models.CharField(max_length=100)
    widget_type = models.CharField(max_length=20, choices=WIDGET_TYPE)
    
    # Target users
    target_user_type = models.CharField(max_length=20, choices=USER_TYPE)
    
    # Data source
    module_name = models.CharField(max_length=100)
    data_function = models.CharField(max_length=200)  # Function to fetch data
    
    # Display
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    icon = models.CharField(max_length=50, blank=True)
    color = models.CharField(max_length=20, blank=True)
    
    # Sizing
    width = models.IntegerField(default=1)  # Grid columns
    height = models.IntegerField(default=1)  # Grid rows
    
    # Refresh
    refresh_interval = models.IntegerField(default=300)  # seconds
    
    is_active = models.BooleanField(default=True)
    sort_order = models.IntegerField(default=0)

class UserDashboardPreference(models.Model):
    """User dashboard preferences"""
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='dashboard_preference')
    
    # Layout
    layout = models.TextField(blank=True)  # JSON layout configuration
    
    # Widget visibility
    visible_widgets = models.TextField(blank=True)  # JSON list of widget IDs
    hidden_widgets = models.TextField(blank=True)  # JSON list of widget IDs
    
    # Theme
    theme = models.CharField(max_length=20, default='light')
    
    updated_at = models.DateTimeField(auto_now=True)

class WidgetPosition(models.Model):
    """User's widget positions"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='widget_positions')
    widget = models.ForeignKey(DashboardWidget, on_delete=models.CASCADE)
    
    position_x = models.IntegerField(default=0)
    position_y = models.IntegerField(default=0)
    width = models.IntegerField(null=True, blank=True)  # Override default
    height = models.IntegerField(null=True, blank=True)
    
    is_visible = models.BooleanField(default=True)
    
    class Meta:
        unique_together = ['user', 'widget']

# ==================== ANNOUNCEMENTS ====================

class DashboardAnnouncement(models.Model):
    """System-wide announcements"""
    PRIORITY = [
        ('LOW', 'Low'),
        ('MEDIUM', 'Medium'),
        ('HIGH', 'High'),
        ('CRITICAL', 'Critical'),
    ]
    
    TARGET_TYPE = [
        ('ALL', 'All Users'),
        ('STUDENT', 'Students'),
        ('FACULTY', 'Faculty'),
        ('STAFF', 'Staff'),
        ('DEPARTMENT', 'Department'),
    ]
    
    title = models.CharField(max_length=300)
    content = models.TextField()
    
    priority = models.CharField(max_length=20, choices=PRIORITY, default='MEDIUM')
    target_type = models.CharField(max_length=20, choices=TARGET_TYPE, default='ALL')
    target_department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    # Visibility
    start_date = models.DateTimeField()
    end_date = models.DateTimeField()
    
    # Link
    action_link = models.URLField(blank=True)
    action_text = models.CharField(max_length=50, blank=True)
    
    posted_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    is_active = models.BooleanField(default=True)

class AnnouncementDismissal(models.Model):
    """Track dismissed announcements"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    announcement = models.ForeignKey(DashboardAnnouncement, on_delete=models.CASCADE)
    dismissed_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['user', 'announcement']

# ==================== QUICK LINKS ====================

class QuickLink(models.Model):
    """Quick access links"""
    USER_TYPE = [
        ('student', 'Student'),
        ('faculty', 'Faculty'),
        ('staff', 'Staff'),
        ('all', 'All'),
    ]
    
    title = models.CharField(max_length=100)
    url = models.CharField(max_length=500)
    icon = models.CharField(max_length=50, blank=True)
    
    target_user_type = models.CharField(max_length=20, choices=USER_TYPE)
    module = models.CharField(max_length=100, blank=True)
    
    sort_order = models.IntegerField(default=0)
    is_active = models.BooleanField(default=True)

class UserQuickLink(models.Model):
    """User's personal quick links"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='personal_quick_links')
    
    title = models.CharField(max_length=100)
    url = models.CharField(max_length=500)
    icon = models.CharField(max_length=50, blank=True)
    
    sort_order = models.IntegerField(default=0)
    
    class Meta:
        ordering = ['sort_order']

# ==================== RECENT ACTIVITY ====================

class UserActivity(models.Model):
    """Track user's recent activities"""
    ACTIVITY_TYPE = [
        ('VIEW', 'Viewed'),
        ('CREATE', 'Created'),
        ('UPDATE', 'Updated'),
        ('SUBMIT', 'Submitted'),
        ('APPROVE', 'Approved'),
    ]
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='recent_activities')
    
    activity_type = models.CharField(max_length=20, choices=ACTIVITY_TYPE)
    module = models.CharField(max_length=100)
    
    title = models.CharField(max_length=300)
    description = models.TextField(blank=True)
    
    # Reference
    object_type = models.CharField(max_length=100, blank=True)
    object_id = models.CharField(max_length=100, blank=True)
    link = models.CharField(max_length=500, blank=True)
    
    timestamp = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['-timestamp']

# ==================== SAVED REPORTS ====================

class SavedReport(models.Model):
    """User saved reports/queries"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='saved_reports')
    
    name = models.CharField(max_length=200)
    module = models.CharField(max_length=100)
    
    # Report configuration
    report_type = models.CharField(max_length=100)
    parameters = models.TextField(blank=True)  # JSON
    
    created_at = models.DateTimeField(auto_now_add=True)
    last_accessed = models.DateTimeField(auto_now=True)

# ==================== CALENDAR EVENTS ====================

class DashboardCalendarEvent(models.Model):
    """Calendar events for dashboard"""
    EVENT_TYPE = [
        ('ACADEMIC', 'Academic'),
        ('EXAMINATION', 'Examination'),
        ('HOLIDAY', 'Holiday'),
        ('EVENT', 'Event'),
        ('DEADLINE', 'Deadline'),
        ('PERSONAL', 'Personal'),
    ]
    
    title = models.CharField(max_length=300)
    description = models.TextField(blank=True)
    event_type = models.CharField(max_length=20, choices=EVENT_TYPE)
    
    start_date = models.DateTimeField()
    end_date = models.DateTimeField(null=True, blank=True)
    all_day = models.BooleanField(default=False)
    
    # Source
    source_module = models.CharField(max_length=100, blank=True)
    source_id = models.CharField(max_length=100, blank=True)
    
    # Target
    is_public = models.BooleanField(default=False)
    user = models.ForeignKey(User, on_delete=models.CASCADE, null=True, blank=True)  # For personal events
    
    color = models.CharField(max_length=20, blank=True)
```

---

## Dashboard Data Functions

### Student Dashboard Data
```python
def get_student_dashboard_data(user):
    """Get all dashboard data for a student"""
    from applications.globals.models import ExtraInfo
    from applications.academic_information.models import Student
    from applications.online_cms.models import Student_grades
    from applications.academic_procedures.models import course_registration, Dues
    
    extrainfo = ExtraInfo.objects.get(user=user)
    student = Student.objects.get(id=extrainfo)
    
    dashboard_data = {
        'profile': {
            'name': f"{user.first_name} {user.last_name}",
            'roll_no': extrainfo.id,
            'batch': str(student.batch_id),
        },
        
        # Current courses
        'registered_courses': course_registration.objects.filter(
            student_id=student,
            # current semester filter
        ).count(),
        
        # Pending items
        'pending_assignments': Assignment.objects.filter(
            # student filter and pending
        ).count(),
        
        'pending_quizzes': Quiz.objects.filter(
            # student filter and upcoming
        ).count(),
        
        # Dues - calculate from individual due fields
        'total_dues': 0,
    }
    
    # Get dues
    try:
        dues = Dues.objects.get(student_id=student)
        dashboard_data['total_dues'] = (
            dues.mess_due + dues.hostel_due + dues.library_due +
            dues.placement_cell_due + dues.academic_due
        )
    except Dues.DoesNotExist:
        pass
    
    # Attendance (if tracked)
    dashboard_data['attendance_percentage'] = 0  # Calculate from attendance records
    
    return dashboard_data
```

### Faculty Dashboard Data
```python
def get_faculty_dashboard_data(user):
    """Get all dashboard data for faculty"""
    from applications.globals.models import ExtraInfo, Faculty
    from applications.programme_curriculum.models import CourseInstructor
    from applications.online_cms.models import Assignment, Quiz
    from applications.leave.models import LeaveApplication  # if exists
    
    extrainfo = ExtraInfo.objects.get(user=user)
    faculty = Faculty.objects.get(id=extrainfo)
    
    # Get courses taught this semester
    current_courses = CourseInstructor.objects.filter(
        instructor_id=extrainfo,
        # current year filter
    )
    
    dashboard_data = {
        'profile': {
            'name': f"{user.first_name} {user.last_name}",
            'department': extrainfo.department.name if extrainfo.department else '',
        },
        
        # Teaching
        'courses_this_semester': current_courses.count(),
        'total_students': 0,  # Sum of enrolled students
        
        # Pending items
        'assignments_to_grade': Assignment.objects.filter(
            # faculty's courses, submitted but not graded
        ).count(),
        
        'pending_leave_approvals': 0,  # If faculty is HOD
        
        # Research (from RSPC if integrated)
        'active_projects': 0,
        'publications': 0,
    }
    
    return dashboard_data
```

### Admin Dashboard Data
```python
def get_admin_dashboard_data(user):
    """Get all dashboard data for administrators"""
    from django.contrib.auth.models import User
    from applications.globals.models import ExtraInfo
    from applications.academic_information.models import Student
    
    dashboard_data = {
        # User statistics
        'total_users': User.objects.filter(is_active=True).count(),
        'total_students': ExtraInfo.objects.filter(user_type='student').count(),
        'total_faculty': ExtraInfo.objects.filter(user_type='faculty').count(),
        'total_staff': ExtraInfo.objects.filter(user_type='staff').count(),
        
        # System health
        'pending_complaints': 0,  # From complaint module
        'pending_approvals': 0,  # Various pending approvals
        
        # Module-wise pending items
        'module_pending': {
            'hostel': 0,
            'mess': 0,
            'placement': 0,
        },
    }
    
    return dashboard_data
```

### Get Upcoming Deadlines
```python
def get_upcoming_deadlines(user, days=7):
    """Get upcoming deadlines for user"""
    from datetime import datetime, timedelta
    from django.utils import timezone
    
    now = timezone.now()
    end_date = now + timedelta(days=days)
    
    deadlines = []
    
    extrainfo = ExtraInfo.objects.get(user=user)
    
    if extrainfo.user_type == 'student':
        # Assignment deadlines
        from applications.online_cms.models import Assignment
        assignments = Assignment.objects.filter(
            deadline__gte=now,
            deadline__lte=end_date,
            # student's courses
        )
        for a in assignments:
            deadlines.append({
                'title': f"Assignment: {a.name}",
                'deadline': a.deadline,
                'module': 'LMS',
                'link': f"/cms/assignment/{a.id}/"
            })
        
        # Quiz deadlines
        from applications.online_cms.models import Quiz
        quizzes = Quiz.objects.filter(
            end_time__gte=now,
            end_time__lte=end_date,
        )
        for q in quizzes:
            deadlines.append({
                'title': f"Quiz: {q.description[:50]}",
                'deadline': q.end_time,
                'module': 'LMS',
                'link': f"/cms/quiz/{q.id}/"
            })
    
    return sorted(deadlines, key=lambda x: x['deadline'])
```

### Get Recent Notifications
```python
def get_recent_notifications(user, limit=10):
    """Get recent notifications for user"""
    from applications.notifications_extension.models import Notification  # if exists
    
    # Query notification model
    notifications = []
    
    # This would integrate with the notification module
    
    return notifications[:limit]
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/dashboard/api/data/` | GET | Dashboard data |
| `/dashboard/api/widgets/` | GET | Available widgets |
| `/dashboard/api/preferences/` | GET, PUT | User preferences |
| `/dashboard/api/announcements/` | GET | Announcements |
| `/dashboard/api/quick-links/` | GET, POST | Quick links |
| `/dashboard/api/calendar/` | GET | Calendar events |
| `/dashboard/api/deadlines/` | GET | Upcoming deadlines |

---

## Important Sync Points

1. **User Type**: Check `ExtraInfo.user_type` to determine dashboard type.
2. **Role-Based Data**: Different widgets for different user types.
3. **Real-Time**: Poll or WebSocket for live updates.
4. **Cross-Module**: Aggregate data from all Fusion modules.
5. **Notifications**: Integrate with notification extension.

---

