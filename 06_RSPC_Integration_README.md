# RSPC (Research, Sponsored Projects and Consultancy) Module - Integration Guide

**What this module is:** The RSPC module will manage research activities - sponsored projects (funded research), consultancy services, patent filings, publications, and research funding applications.

**Why we need to build this:** Faculty and students conduct research that needs tracking. Currently sponsored projects are managed in spreadsheets with budgets, timelines, and deliverables tracked manually. This module will provide unified project lifecycle management for the RSPC cell.

**Why integration is needed:** Research is conducted by Faculty (from globals), may involve PhD students (from Student), belongs to disciplines/departments (from Discipline), and funding often requires student/faculty verification. All identity and organizational data comes from core modules.

**Key dependencies:** Faculty, Student, Discipline, ExtraInfo

---

## Module Overview
The RSPC module manages research activities, sponsored projects, consultancy services, patent filings, publications tracking, and research funding for faculty and students.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Programme`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `id` | AutoField | Primary Key | Programme reference |
| `category` | CharField(3) | UG/PG/PHD | Research eligibility (PhD scholars) |
| `name` | CharField(70) | Programme name | Display purposes |

#### Table: `Discipline`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `id` | AutoField | Primary Key | Discipline/Department reference |
| `name` | CharField(100) | Discipline name | **Research area mapping** |
| `acronym` | CharField(10) | Short code | Display purposes |

#### Table: `Course`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `id` | AutoField | Primary Key | Course reference |
| `code` | CharField(10) | Course code | Research methodology courses |
| `name` | CharField(100) | Course name | Display purposes |
| `disciplines` | ManyToManyField | Linked disciplines | Department research courses |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **PhD/MTech researcher identification** |
| `programme` | CharField(10) | Programme | Filter research students |
| `batch_id` | ForeignKey(Batch) | Batch | Batch reference |
| `specialization` | CharField(40) | Specialization | Research area |

---

### 3. From `academic_procedures` Module

#### Table: `Thesis`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `student_id` | ForeignKey(Student) | Student | Research student |
| `supervisor_id` | ForeignKey(Faculty) | Supervisor | **Research supervisor** |
| `topic` | CharField(1000) | Topic | **Research topic** |

#### Table: `ThesisTopicProcess`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `student_id` | ForeignKey(Student) | Student | Research student |
| `research_area` | CharField(50) | Research area | **Research domain** |
| `thesis_topic` | CharField(1000) | Topic | Research topic |
| `supervisor_id` | ForeignKey(Faculty) | Supervisor | Primary guide |
| `co_supervisor_id` | ForeignKey(Faculty) | Co-supervisor | Co-guide |
| `member1/2/3` | ForeignKey(Faculty) | Committee | **Research committee** |

---

### 4. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `id` | CharField(20) | User ID | Researcher ID |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | faculty/student |
| `department` | ForeignKey(DepartmentInfo) | Department | **Research department** |

#### Table: `Faculty`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Principal Investigator (PI)** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | Research department |

#### Table: `Designation`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `id` | AutoField | Primary Key | Designation reference |
| `name` | CharField(50) | Designation | Dean RSPC, etc. |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in RSPC |
|--------|------|-------------|---------------|
| `user` | ForeignKey(User) | User | Designation holder |
| `designation` | ForeignKey(Designation) | Designation | **Dean RSPC identification** |

---

## Data Models to Create in RSPC Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, Faculty, DepartmentInfo
from applications.academic_information.models import Student
from applications.programme_curriculum.models import Discipline

# ==================== RESEARCH AREAS ====================

class ResearchGroup(models.Model):
    """Research groups/labs in the institute"""
    name = models.CharField(max_length=200)
    acronym = models.CharField(max_length=20, blank=True)
    discipline = models.ForeignKey(Discipline, on_delete=models.CASCADE)
    description = models.TextField()
    head = models.ForeignKey(Faculty, on_delete=models.SET_NULL, null=True, related_name='led_groups')
    members = models.ManyToManyField(Faculty, related_name='research_groups')
    established_date = models.DateField(null=True)
    website = models.URLField(blank=True)
    is_active = models.BooleanField(default=True)

class ResearchArea(models.Model):
    """Research areas and specializations"""
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    parent_area = models.ForeignKey('self', on_delete=models.SET_NULL, null=True, blank=True)
    discipline = models.ForeignKey(Discipline, on_delete=models.CASCADE)
    faculty_experts = models.ManyToManyField(Faculty, related_name='expertise_areas')
    is_active = models.BooleanField(default=True)

# ==================== SPONSORED PROJECTS ====================

class FundingAgency(models.Model):
    """Funding agencies (SERB, DST, DRDO, etc.)"""
    AGENCY_TYPE = [
        ('GOVERNMENT', 'Government'),
        ('PRIVATE', 'Private'),
        ('INTERNATIONAL', 'International'),
        ('INDUSTRY', 'Industry'),
        ('OTHER', 'Other'),
    ]
    
    name = models.CharField(max_length=200)
    acronym = models.CharField(max_length=20, blank=True)
    agency_type = models.CharField(max_length=20, choices=AGENCY_TYPE)
    country = models.CharField(max_length=50, default='India')
    website = models.URLField(blank=True)
    contact_info = models.TextField(blank=True)
    is_active = models.BooleanField(default=True)

class SponsoredProject(models.Model):
    """Sponsored research projects"""
    PROJECT_STATUS = [
        ('PROPOSED', 'Proposed'),
        ('SUBMITTED', 'Submitted'),
        ('UNDER_REVIEW', 'Under Review'),
        ('SANCTIONED', 'Sanctioned'),
        ('ONGOING', 'Ongoing'),
        ('EXTENDED', 'Extended'),
        ('COMPLETED', 'Completed'),
        ('TERMINATED', 'Terminated'),
    ]
    
    # Basic info
    title = models.CharField(max_length=500)
    project_number = models.CharField(max_length=50, unique=True)
    description = models.TextField()
    research_area = models.ForeignKey(ResearchArea, on_delete=models.SET_NULL, null=True)
    
    # Team
    principal_investigator = models.ForeignKey(Faculty, on_delete=models.PROTECT, related_name='pi_projects')
    co_principal_investigators = models.ManyToManyField(Faculty, related_name='copi_projects', blank=True)
    research_scholars = models.ManyToManyField(Student, related_name='sponsored_projects', blank=True)
    
    # Funding
    funding_agency = models.ForeignKey(FundingAgency, on_delete=models.PROTECT)
    sanctioned_amount = models.DecimalField(max_digits=15, decimal_places=2)
    utilized_amount = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    
    # Timeline
    submission_date = models.DateField(null=True, blank=True)
    sanction_date = models.DateField(null=True, blank=True)
    start_date = models.DateField(null=True, blank=True)
    original_end_date = models.DateField(null=True, blank=True)
    extended_end_date = models.DateField(null=True, blank=True)
    actual_end_date = models.DateField(null=True, blank=True)
    
    # Status
    status = models.CharField(max_length=20, choices=PROJECT_STATUS, default='PROPOSED')
    
    # Files
    proposal_document = models.FileField(upload_to='rspc/projects/proposals/', null=True, blank=True)
    sanction_letter = models.FileField(upload_to='rspc/projects/sanctions/', null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

class ProjectExpenditure(models.Model):
    """Project expenditure tracking"""
    EXPENDITURE_HEAD = [
        ('MANPOWER', 'Manpower'),
        ('EQUIPMENT', 'Equipment'),
        ('CONSUMABLES', 'Consumables'),
        ('TRAVEL', 'Travel'),
        ('CONTINGENCY', 'Contingency'),
        ('OVERHEAD', 'Overhead'),
        ('OTHER', 'Other'),
    ]
    
    project = models.ForeignKey(SponsoredProject, on_delete=models.CASCADE, related_name='expenditures')
    expenditure_head = models.CharField(max_length=20, choices=EXPENDITURE_HEAD)
    description = models.TextField()
    amount = models.DecimalField(max_digits=15, decimal_places=2)
    date = models.DateField()
    voucher_number = models.CharField(max_length=50, blank=True)
    bill_document = models.FileField(upload_to='rspc/projects/bills/', null=True, blank=True)
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True)
    created_at = models.DateTimeField(auto_now_add=True)

class ProjectMilestone(models.Model):
    """Project milestones and deliverables"""
    project = models.ForeignKey(SponsoredProject, on_delete=models.CASCADE, related_name='milestones')
    title = models.CharField(max_length=200)
    description = models.TextField()
    due_date = models.DateField()
    completed_date = models.DateField(null=True, blank=True)
    is_completed = models.BooleanField(default=False)
    deliverables = models.TextField(blank=True)
    
class ProjectReport(models.Model):
    """Project progress reports"""
    REPORT_TYPE = [
        ('QUARTERLY', 'Quarterly'),
        ('HALF_YEARLY', 'Half Yearly'),
        ('ANNUAL', 'Annual'),
        ('FINAL', 'Final'),
        ('UTILIZATION', 'Utilization Certificate'),
    ]
    
    project = models.ForeignKey(SponsoredProject, on_delete=models.CASCADE, related_name='reports')
    report_type = models.CharField(max_length=20, choices=REPORT_TYPE)
    period_from = models.DateField()
    period_to = models.DateField()
    summary = models.TextField()
    report_file = models.FileField(upload_to='rspc/projects/reports/')
    submitted_date = models.DateField()
    approved = models.BooleanField(default=False)

# ==================== CONSULTANCY ====================

class ConsultancyProject(models.Model):
    """Consultancy projects"""
    PROJECT_STATUS = [
        ('PROPOSED', 'Proposed'),
        ('NEGOTIATION', 'Under Negotiation'),
        ('APPROVED', 'Approved'),
        ('ONGOING', 'Ongoing'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    # Basic info
    title = models.CharField(max_length=500)
    project_number = models.CharField(max_length=50, unique=True)
    description = models.TextField()
    
    # Client
    client_name = models.CharField(max_length=200)
    client_type = models.CharField(max_length=50)  # Industry, Government, etc.
    client_contact = models.TextField()
    
    # Team
    consultant = models.ForeignKey(Faculty, on_delete=models.PROTECT, related_name='consultancies')
    co_consultants = models.ManyToManyField(Faculty, related_name='co_consultancies', blank=True)
    
    # Financial
    contract_amount = models.DecimalField(max_digits=15, decimal_places=2)
    faculty_share = models.DecimalField(max_digits=15, decimal_places=2)
    institute_share = models.DecimalField(max_digits=15, decimal_places=2)
    
    # Timeline
    start_date = models.DateField()
    end_date = models.DateField()
    
    # Status
    status = models.CharField(max_length=20, choices=PROJECT_STATUS, default='PROPOSED')
    
    # Documents
    agreement_document = models.FileField(upload_to='rspc/consultancy/agreements/', null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)

# ==================== PUBLICATIONS ====================

class Publication(models.Model):
    """Research publications"""
    PUBLICATION_TYPE = [
        ('JOURNAL', 'Journal Article'),
        ('CONFERENCE', 'Conference Paper'),
        ('BOOK', 'Book'),
        ('BOOK_CHAPTER', 'Book Chapter'),
        ('PATENT', 'Patent'),
        ('THESIS', 'Thesis'),
        ('TECHNICAL_REPORT', 'Technical Report'),
        ('OTHER', 'Other'),
    ]
    
    INDEX_TYPE = [
        ('SCI', 'SCI'),
        ('SCIE', 'SCIE'),
        ('SCOPUS', 'Scopus'),
        ('WOS', 'Web of Science'),
        ('UGCCL', 'UGC Care List'),
        ('OTHER', 'Other'),
        ('NONE', 'Non-indexed'),
    ]
    
    # Basic info
    title = models.CharField(max_length=500)
    publication_type = models.CharField(max_length=20, choices=PUBLICATION_TYPE)
    
    # Authors (from institute)
    faculty_authors = models.ManyToManyField(Faculty, related_name='publications')
    student_authors = models.ManyToManyField(Student, related_name='publications', blank=True)
    external_authors = models.TextField(blank=True)  # Names of external authors
    
    # Publication details
    journal_conference_name = models.CharField(max_length=300)
    publisher = models.CharField(max_length=200, blank=True)
    volume = models.CharField(max_length=20, blank=True)
    issue = models.CharField(max_length=20, blank=True)
    pages = models.CharField(max_length=20, blank=True)
    year = models.IntegerField()
    month = models.IntegerField(null=True, blank=True)
    
    # Indexing
    index_type = models.CharField(max_length=20, choices=INDEX_TYPE, default='NONE')
    impact_factor = models.DecimalField(max_digits=6, decimal_places=3, null=True, blank=True)
    
    # Identifiers
    doi = models.CharField(max_length=100, blank=True)
    issn = models.CharField(max_length=20, blank=True)
    isbn = models.CharField(max_length=20, blank=True)
    url = models.URLField(blank=True)
    
    # File
    pdf_file = models.FileField(upload_to='rspc/publications/', null=True, blank=True)
    
    # Verification
    is_verified = models.BooleanField(default=False)
    verified_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)

# ==================== PATENTS ====================

class Patent(models.Model):
    """Patent applications and grants"""
    PATENT_STATUS = [
        ('DRAFT', 'Draft'),
        ('FILED', 'Filed'),
        ('PUBLISHED', 'Published'),
        ('UNDER_EXAMINATION', 'Under Examination'),
        ('GRANTED', 'Granted'),
        ('REJECTED', 'Rejected'),
        ('LAPSED', 'Lapsed'),
    ]
    
    PATENT_TYPE = [
        ('NATIONAL', 'National'),
        ('INTERNATIONAL', 'International'),
        ('PCT', 'PCT'),
    ]
    
    # Basic info
    title = models.CharField(max_length=500)
    abstract = models.TextField()
    patent_type = models.CharField(max_length=20, choices=PATENT_TYPE)
    
    # Inventors
    faculty_inventors = models.ManyToManyField(Faculty, related_name='patents')
    student_inventors = models.ManyToManyField(Student, related_name='patents', blank=True)
    external_inventors = models.TextField(blank=True)
    
    # Filing details
    application_number = models.CharField(max_length=50, blank=True)
    filing_date = models.DateField(null=True, blank=True)
    publication_number = models.CharField(max_length=50, blank=True)
    publication_date = models.DateField(null=True, blank=True)
    grant_number = models.CharField(max_length=50, blank=True)
    grant_date = models.DateField(null=True, blank=True)
    
    # Status
    status = models.CharField(max_length=20, choices=PATENT_STATUS, default='DRAFT')
    
    # Documents
    specification_document = models.FileField(upload_to='rspc/patents/specs/', null=True, blank=True)
    grant_certificate = models.FileField(upload_to='rspc/patents/certificates/', null=True, blank=True)
    
    # Related project (if any)
    related_project = models.ForeignKey(SponsoredProject, on_delete=models.SET_NULL, null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)

# ==================== RESEARCH SCHOLARS ====================

class ResearchScholar(models.Model):
    """Extended information for research scholars"""
    student = models.OneToOneField(Student, on_delete=models.CASCADE, primary_key=True)
    
    # Supervisor info (linked from ThesisTopicProcess)
    enrollment_date = models.DateField()
    expected_completion = models.DateField(null=True, blank=True)
    
    # Fellowship
    fellowship_type = models.CharField(max_length=50, blank=True)  # GATE, NET, Institute, etc.
    fellowship_amount = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    # Progress
    coursework_completed = models.BooleanField(default=False)
    comprehensive_exam_passed = models.BooleanField(default=False)
    comprehensive_exam_date = models.DateField(null=True, blank=True)
    
    # Synopsis
    synopsis_submitted = models.BooleanField(default=False)
    synopsis_date = models.DateField(null=True, blank=True)
    
    # Thesis
    thesis_submitted = models.BooleanField(default=False)
    thesis_submission_date = models.DateField(null=True, blank=True)
    defense_date = models.DateField(null=True, blank=True)
    degree_awarded_date = models.DateField(null=True, blank=True)
```

---

## Integration Functions

### Get Faculty Research Profile
```python
def get_faculty_research_profile(faculty_id):
    """Get comprehensive research profile of faculty"""
    from applications.globals.models import Faculty
    from applications.academic_procedures.models import ThesisTopicProcess
    
    faculty = Faculty.objects.get(id=faculty_id)
    extrainfo = faculty.id
    
    # Get supervised students
    supervised_students = ThesisTopicProcess.objects.filter(
        supervisor_id=faculty,
        approval_supervisor=True
    ).select_related('student_id')
    
    # Get publications
    publications = Publication.objects.filter(faculty_authors=faculty)
    
    # Get patents
    patents = Patent.objects.filter(faculty_inventors=faculty)
    
    # Get projects
    pi_projects = SponsoredProject.objects.filter(principal_investigator=faculty)
    copi_projects = SponsoredProject.objects.filter(co_principal_investigators=faculty)
    
    # Get consultancies
    consultancies = ConsultancyProject.objects.filter(consultant=faculty)
    
    return {
        'faculty': faculty,
        'extrainfo': extrainfo,
        'supervised_students': list(supervised_students),
        'publications': {
            'total': publications.count(),
            'journals': publications.filter(publication_type='JOURNAL').count(),
            'conferences': publications.filter(publication_type='CONFERENCE').count(),
            'sci_indexed': publications.filter(index_type__in=['SCI', 'SCIE']).count(),
        },
        'patents': {
            'total': patents.count(),
            'granted': patents.filter(status='GRANTED').count(),
        },
        'projects': {
            'as_pi': pi_projects.count(),
            'as_copi': copi_projects.count(),
            'ongoing': pi_projects.filter(status='ONGOING').count(),
            'total_funding': sum(p.sanctioned_amount for p in pi_projects),
        },
        'consultancies': {
            'total': consultancies.count(),
            'ongoing': consultancies.filter(status='ONGOING').count(),
        },
    }
```

### Link Research to Thesis
```python
def link_thesis_to_research(thesis_process_id):
    """Link thesis topic to RSPC research tracking"""
    from applications.academic_procedures.models import ThesisTopicProcess
    
    thesis = ThesisTopicProcess.objects.get(id=thesis_process_id)
    
    # Create or get research scholar entry
    scholar, created = ResearchScholar.objects.get_or_create(
        student=thesis.student_id,
        defaults={
            'enrollment_date': thesis.date,
        }
    )
    
    # Find or create research area
    research_area, _ = ResearchArea.objects.get_or_create(
        name=thesis.research_area,
        defaults={
            'discipline': thesis.student_id.batch_id.discipline,
        }
    )
    
    return scholar, research_area
```

### Get Department Research Statistics
```python
def get_department_research_stats(department_id):
    """Get research statistics for a department"""
    from applications.globals.models import DepartmentInfo, ExtraInfo, Faculty
    from applications.programme_curriculum.models import Discipline
    
    department = DepartmentInfo.objects.get(id=department_id)
    
    # Get faculty in department
    faculty_extrainfos = ExtraInfo.objects.filter(
        department=department,
        user_type='faculty'
    )
    
    faculty_list = Faculty.objects.filter(id__in=faculty_extrainfos)
    
    # Get discipline
    discipline = Discipline.objects.filter(
        name__icontains=department.name.replace('department:', '').strip()
    ).first()
    
    # Count publications
    total_publications = Publication.objects.filter(
        faculty_authors__in=faculty_list
    ).distinct().count()
    
    # Count projects
    total_projects = SponsoredProject.objects.filter(
        principal_investigator__in=faculty_list
    ).count()
    
    ongoing_projects = SponsoredProject.objects.filter(
        principal_investigator__in=faculty_list,
        status='ONGOING'
    )
    
    total_funding = sum(p.sanctioned_amount for p in ongoing_projects)
    
    # Count patents
    total_patents = Patent.objects.filter(
        faculty_inventors__in=faculty_list
    ).distinct().count()
    
    granted_patents = Patent.objects.filter(
        faculty_inventors__in=faculty_list,
        status='GRANTED'
    ).distinct().count()
    
    return {
        'department': department,
        'faculty_count': faculty_list.count(),
        'publications': total_publications,
        'projects': total_projects,
        'ongoing_funding': total_funding,
        'patents_total': total_patents,
        'patents_granted': granted_patents,
    }
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/rspc/api/projects/` | GET, POST | Sponsored projects |
| `/rspc/api/projects/<id>/expenditures/` | GET, POST | Project expenditures |
| `/rspc/api/consultancies/` | GET, POST | Consultancy projects |
| `/rspc/api/publications/` | GET, POST | Publications |
| `/rspc/api/patents/` | GET, POST | Patents |
| `/rspc/api/research-areas/` | GET | Research areas |
| `/rspc/api/funding-agencies/` | GET | Funding agencies |
| `/rspc/api/faculty/<id>/profile/` | GET | Faculty research profile |
| `/rspc/api/department/<id>/stats/` | GET | Department stats |
| `/rspc/api/scholars/` | GET | Research scholars |

---

## Important Sync Points

1. **Faculty Data**: All PI/Consultant info comes from `globals.Faculty`.

2. **Student Researchers**: PhD/MTech students from `academic_information.Student`.

3. **Thesis Linkage**: Connect with `academic_procedures.ThesisTopicProcess` for research tracking.

4. **Department Stats**: Use `globals.DepartmentInfo` and `programme_curriculum.Discipline`.

5. **File Tracking**: Integrate with FTS for project approvals and document routing.

---

