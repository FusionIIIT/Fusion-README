# P&S Management (Purchase and Store Management) Module - Integration Guide

**What this module is:** The Purchase and Store Management module will handle procurement - purchase requests (indents), vendor management, purchase orders, goods receipt, inventory tracking, and store operations.

**Why we need to build this:** Institutes purchase equipment, consumables, and services. Currently indents are paper-based, vendor records scattered. This module will digitize the entire procurement workflow - from indent to goods receipt - with proper approvals and inventory tracking.

**Why integration is needed:** Purchase requests come from departments (Discipline) and are raised by employees (ExtraInfo/Faculty/Staff). Budget heads may be linked to projects (RSPC). Approvals route through organizational hierarchy. The module needs organizational structure data from core modules.

**Key dependencies:** Discipline, ExtraInfo, Faculty, Staff, DepartmentInfo

---

## Module Overview
The Purchase and Store Management module handles procurement processes, inventory management, vendor management, purchase orders, goods receipt, and store operations for the institute.

---

## Core Dependencies - Tables to Sync With

### 1. From `programme_curriculum` Module

#### Table: `Discipline`
| Column | Type | Description | Usage in P&S |
|--------|------|-------------|--------------|
| `id` | AutoField | Primary Key | Department reference for indent |
| `name` | CharField(100) | Discipline name | Indent department |
| `acronym` | CharField(10) | Short code | Quick reference |

---

### 2. From `academic_information` Module

#### Table: `Student`
| Column | Type | Description | Usage in P&S |
|--------|------|-------------|--------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | Lab equipment requisitions |
| `batch_id` | ForeignKey(Batch) | Batch | Department context |

---

### 3. From `academic_procedures` Module

#### Table: `SponsoredProject` (from RSPC - if integrated)
| Column | Type | Description | Usage in P&S |
|--------|------|-------------|--------------|
| `project_number` | CharField(50) | Project ID | **Project-linked purchases** |
| `sanctioned_amount` | DecimalField | Budget | Budget verification |

---

### 4. From `globals` Module

#### Table: `ExtraInfo`
| Column | Type | Description | Usage in P&S |
|--------|------|-------------|--------------|
| `id` | CharField(20) | User ID | **Indenter identification** |
| `user` | OneToOneField(User) | User | User reference |
| `user_type` | CharField(20) | Type | Role filtering |
| `department` | ForeignKey(DepartmentInfo) | Department | **Indent department** |

#### Table: `Faculty`
| Column | Type | Description | Usage in P&S |
|--------|------|-------------|--------------|
| `id` | OneToOneField(ExtraInfo) | Primary Key | **Indenter (faculty)** |

#### Table: `DepartmentInfo`
| Column | Type | Description | Usage in P&S |
|--------|------|-------------|--------------|
| `id` | AutoField | Primary Key | Department reference |
| `name` | CharField(100) | Department name | **Indent routing** |

#### Table: `Designation`
| Column | Type | Description | Usage in P&S |
|--------|------|-------------|--------------|
| `id` | AutoField | Primary Key | Designation reference |
| `name` | CharField(50) | Designation | **Approval authority** |

#### Table: `HoldsDesignation`
| Column | Type | Description | Usage in P&S |
|--------|------|-------------|--------------|
| `user` | ForeignKey(User) | User | Designation holder |
| `designation` | ForeignKey(Designation) | Designation | **HOD/Dean approval** |

---

## Data Models to Create in P&S Module

```python
from django.db import models
from django.contrib.auth.models import User
from applications.globals.models import ExtraInfo, Faculty, DepartmentInfo, Designation

# ==================== VENDOR MANAGEMENT ====================

class Vendor(models.Model):
    """Registered vendors"""
    VENDOR_TYPE = [
        ('SUPPLIER', 'Supplier'),
        ('MANUFACTURER', 'Manufacturer'),
        ('DISTRIBUTOR', 'Distributor'),
        ('SERVICE_PROVIDER', 'Service Provider'),
    ]
    
    STATUS = [
        ('ACTIVE', 'Active'),
        ('INACTIVE', 'Inactive'),
        ('BLACKLISTED', 'Blacklisted'),
    ]
    
    name = models.CharField(max_length=200)
    vendor_type = models.CharField(max_length=20, choices=VENDOR_TYPE)
    registration_number = models.CharField(max_length=50, unique=True)
    
    # Contact
    address = models.TextField()
    city = models.CharField(max_length=100)
    state = models.CharField(max_length=100)
    pincode = models.CharField(max_length=10)
    contact_person = models.CharField(max_length=100)
    phone = models.CharField(max_length=20)
    email = models.EmailField()
    website = models.URLField(blank=True)
    
    # Tax info
    gst_number = models.CharField(max_length=20, blank=True)
    pan_number = models.CharField(max_length=15, blank=True)
    
    # Bank details
    bank_name = models.CharField(max_length=100, blank=True)
    bank_account = models.CharField(max_length=30, blank=True)
    ifsc_code = models.CharField(max_length=15, blank=True)
    
    # Categories
    product_categories = models.TextField(blank=True)  # JSON list
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='ACTIVE')
    registration_date = models.DateField(auto_now_add=True)
    valid_until = models.DateField(null=True, blank=True)
    
    # Rating
    rating = models.DecimalField(max_digits=3, decimal_places=2, default=0)
    
    class Meta:
        ordering = ['name']

class VendorDocument(models.Model):
    """Vendor registration documents"""
    vendor = models.ForeignKey(Vendor, on_delete=models.CASCADE, related_name='documents')
    document_type = models.CharField(max_length=50)  # GST, PAN, Registration, etc.
    document = models.FileField(upload_to='ps/vendors/documents/')
    uploaded_at = models.DateTimeField(auto_now_add=True)
    valid_until = models.DateField(null=True, blank=True)

# ==================== ITEM MANAGEMENT ====================

class ItemCategory(models.Model):
    """Categories of items"""
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=20, unique=True)
    description = models.TextField(blank=True)
    parent_category = models.ForeignKey('self', on_delete=models.SET_NULL, null=True, blank=True)
    is_active = models.BooleanField(default=True)

class Item(models.Model):
    """Master list of items"""
    ITEM_TYPE = [
        ('CONSUMABLE', 'Consumable'),
        ('NON_CONSUMABLE', 'Non-Consumable'),
        ('EQUIPMENT', 'Equipment'),
        ('FURNITURE', 'Furniture'),
        ('STATIONERY', 'Stationery'),
        ('LAB_EQUIPMENT', 'Lab Equipment'),
        ('IT_EQUIPMENT', 'IT Equipment'),
        ('OTHER', 'Other'),
    ]
    
    UOM = [
        ('NOS', 'Numbers'),
        ('KG', 'Kilograms'),
        ('LTR', 'Liters'),
        ('MTR', 'Meters'),
        ('BOX', 'Box'),
        ('PKT', 'Packet'),
        ('SET', 'Set'),
        ('PAIR', 'Pair'),
    ]
    
    item_code = models.CharField(max_length=30, unique=True)
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    category = models.ForeignKey(ItemCategory, on_delete=models.PROTECT)
    item_type = models.CharField(max_length=20, choices=ITEM_TYPE)
    unit_of_measurement = models.CharField(max_length=10, choices=UOM)
    
    # Specifications
    specifications = models.TextField(blank=True)
    make = models.CharField(max_length=100, blank=True)
    model = models.CharField(max_length=100, blank=True)
    
    # Pricing
    last_purchase_price = models.DecimalField(max_digits=15, decimal_places=2, null=True, blank=True)
    standard_price = models.DecimalField(max_digits=15, decimal_places=2, null=True, blank=True)
    
    # Stock
    minimum_stock = models.IntegerField(default=0)
    reorder_level = models.IntegerField(default=0)
    
    # Status
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

# ==================== INDENT MANAGEMENT ====================

class Indent(models.Model):
    """Purchase indents/requisitions"""
    INDENT_TYPE = [
        ('NORMAL', 'Normal'),
        ('URGENT', 'Urgent'),
        ('PROJECT', 'Project-linked'),
        ('GRANT', 'Grant-linked'),
    ]
    
    STATUS = [
        ('DRAFT', 'Draft'),
        ('SUBMITTED', 'Submitted'),
        ('HOD_APPROVED', 'HOD Approved'),
        ('HOD_REJECTED', 'HOD Rejected'),
        ('FINANCE_APPROVED', 'Finance Approved'),
        ('DIRECTOR_APPROVED', 'Director Approved'),
        ('REJECTED', 'Rejected'),
        ('PO_GENERATED', 'PO Generated'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    # Identification
    indent_number = models.CharField(max_length=50, unique=True)
    indent_type = models.CharField(max_length=20, choices=INDENT_TYPE, default='NORMAL')
    
    # Indenter info (from globals)
    indenter = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT, related_name='indents')
    department = models.ForeignKey(DepartmentInfo, on_delete=models.PROTECT)
    
    # Purpose
    purpose = models.TextField()
    justification = models.TextField(blank=True)
    
    # Budget head
    budget_head = models.CharField(max_length=100, blank=True)
    project_number = models.CharField(max_length=50, blank=True)  # Link to RSPC if project-linked
    
    # Financial
    estimated_cost = models.DecimalField(max_digits=15, decimal_places=2)
    
    # Timeline
    created_at = models.DateTimeField(auto_now_add=True)
    required_by = models.DateField(null=True, blank=True)
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='DRAFT')
    
    # Approvals
    hod_approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='hod_approved_indents')
    hod_approved_at = models.DateTimeField(null=True, blank=True)
    hod_remarks = models.TextField(blank=True)
    
    finance_approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='finance_approved_indents')
    finance_approved_at = models.DateTimeField(null=True, blank=True)
    finance_remarks = models.TextField(blank=True)
    
    director_approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='director_approved_indents')
    director_approved_at = models.DateTimeField(null=True, blank=True)
    director_remarks = models.TextField(blank=True)

class IndentItem(models.Model):
    """Items in an indent"""
    indent = models.ForeignKey(Indent, on_delete=models.CASCADE, related_name='items')
    item = models.ForeignKey(Item, on_delete=models.PROTECT, null=True, blank=True)
    item_description = models.CharField(max_length=500)  # For non-catalog items
    specifications = models.TextField(blank=True)
    quantity = models.IntegerField()
    unit = models.CharField(max_length=20)
    estimated_rate = models.DecimalField(max_digits=15, decimal_places=2)
    estimated_amount = models.DecimalField(max_digits=15, decimal_places=2)
    remarks = models.TextField(blank=True)

# ==================== PURCHASE ORDER ====================

class PurchaseOrder(models.Model):
    """Purchase orders"""
    STATUS = [
        ('DRAFT', 'Draft'),
        ('ISSUED', 'Issued'),
        ('ACKNOWLEDGED', 'Acknowledged'),
        ('PARTIAL_RECEIVED', 'Partially Received'),
        ('COMPLETED', 'Completed'),
        ('CANCELLED', 'Cancelled'),
    ]
    
    # Identification
    po_number = models.CharField(max_length=50, unique=True)
    indent = models.ForeignKey(Indent, on_delete=models.PROTECT, related_name='purchase_orders')
    
    # Vendor
    vendor = models.ForeignKey(Vendor, on_delete=models.PROTECT)
    
    # Amounts
    subtotal = models.DecimalField(max_digits=15, decimal_places=2)
    tax_amount = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    discount = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    total_amount = models.DecimalField(max_digits=15, decimal_places=2)
    
    # Timeline
    po_date = models.DateField()
    delivery_date = models.DateField()
    validity_date = models.DateField()
    
    # Terms
    payment_terms = models.TextField(blank=True)
    delivery_terms = models.TextField(blank=True)
    warranty_terms = models.TextField(blank=True)
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='DRAFT')
    
    # Prepared by
    prepared_by = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT)
    created_at = models.DateTimeField(auto_now_add=True)

class PurchaseOrderItem(models.Model):
    """Items in purchase order"""
    purchase_order = models.ForeignKey(PurchaseOrder, on_delete=models.CASCADE, related_name='items')
    indent_item = models.ForeignKey(IndentItem, on_delete=models.PROTECT)
    quantity = models.IntegerField()
    unit_price = models.DecimalField(max_digits=15, decimal_places=2)
    tax_percent = models.DecimalField(max_digits=5, decimal_places=2, default=0)
    discount_percent = models.DecimalField(max_digits=5, decimal_places=2, default=0)
    total_amount = models.DecimalField(max_digits=15, decimal_places=2)
    
    # Delivery tracking
    quantity_received = models.IntegerField(default=0)
    quantity_pending = models.IntegerField(default=0)

# ==================== GOODS RECEIPT ====================

class GoodsReceipt(models.Model):
    """Goods receipt note (GRN)"""
    STATUS = [
        ('RECEIVED', 'Received'),
        ('QC_PENDING', 'QC Pending'),
        ('QC_PASSED', 'QC Passed'),
        ('QC_FAILED', 'QC Failed'),
        ('STORED', 'Stored'),
    ]
    
    grn_number = models.CharField(max_length=50, unique=True)
    purchase_order = models.ForeignKey(PurchaseOrder, on_delete=models.PROTECT)
    
    # Receipt details
    receipt_date = models.DateField()
    challan_number = models.CharField(max_length=50, blank=True)
    challan_date = models.DateField(null=True, blank=True)
    
    # Received by
    received_by = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT)
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='RECEIVED')
    
    # Remarks
    remarks = models.TextField(blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)

class GoodsReceiptItem(models.Model):
    """Items in GRN"""
    grn = models.ForeignKey(GoodsReceipt, on_delete=models.CASCADE, related_name='items')
    po_item = models.ForeignKey(PurchaseOrderItem, on_delete=models.PROTECT)
    quantity_received = models.IntegerField()
    quantity_accepted = models.IntegerField(default=0)
    quantity_rejected = models.IntegerField(default=0)
    rejection_reason = models.TextField(blank=True)

# ==================== INVENTORY/STORE ====================

class Store(models.Model):
    """Stores/Warehouses"""
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=20, unique=True)
    location = models.CharField(max_length=200)
    department = models.ForeignKey(DepartmentInfo, on_delete=models.SET_NULL, null=True, blank=True)
    store_keeper = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True)
    is_active = models.BooleanField(default=True)

class StockEntry(models.Model):
    """Inventory stock entries"""
    TRANSACTION_TYPE = [
        ('RECEIPT', 'Receipt'),
        ('ISSUE', 'Issue'),
        ('RETURN', 'Return'),
        ('ADJUSTMENT', 'Adjustment'),
        ('TRANSFER', 'Transfer'),
    ]
    
    item = models.ForeignKey(Item, on_delete=models.PROTECT)
    store = models.ForeignKey(Store, on_delete=models.PROTECT)
    transaction_type = models.CharField(max_length=20, choices=TRANSACTION_TYPE)
    quantity = models.IntegerField()  # Positive for in, negative for out
    unit_price = models.DecimalField(max_digits=15, decimal_places=2)
    
    # Reference
    reference_type = models.CharField(max_length=50, blank=True)  # GRN, Issue Slip, etc.
    reference_number = models.CharField(max_length=50, blank=True)
    
    # Details
    transaction_date = models.DateField()
    remarks = models.TextField(blank=True)
    
    # User
    created_by = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT)
    created_at = models.DateTimeField(auto_now_add=True)

class CurrentStock(models.Model):
    """Current stock levels"""
    item = models.ForeignKey(Item, on_delete=models.PROTECT)
    store = models.ForeignKey(Store, on_delete=models.PROTECT)
    quantity = models.IntegerField(default=0)
    average_price = models.DecimalField(max_digits=15, decimal_places=2, default=0)
    last_updated = models.DateTimeField(auto_now=True)
    
    class Meta:
        unique_together = ['item', 'store']

# ==================== ISSUE MANAGEMENT ====================

class IssueSlip(models.Model):
    """Material issue slips"""
    STATUS = [
        ('REQUESTED', 'Requested'),
        ('APPROVED', 'Approved'),
        ('ISSUED', 'Issued'),
        ('REJECTED', 'Rejected'),
    ]
    
    slip_number = models.CharField(max_length=50, unique=True)
    
    # Requester
    requester = models.ForeignKey(ExtraInfo, on_delete=models.PROTECT, related_name='issue_requests')
    department = models.ForeignKey(DepartmentInfo, on_delete=models.PROTECT)
    
    # Purpose
    purpose = models.TextField()
    
    # Store
    store = models.ForeignKey(Store, on_delete=models.PROTECT)
    
    # Status
    status = models.CharField(max_length=20, choices=STATUS, default='REQUESTED')
    
    # Approval
    approved_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='approved_issues')
    approved_at = models.DateTimeField(null=True, blank=True)
    
    # Issue
    issued_by = models.ForeignKey(ExtraInfo, on_delete=models.SET_NULL, null=True, blank=True, related_name='issued_materials')
    issued_at = models.DateTimeField(null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)

class IssueSlipItem(models.Model):
    """Items in issue slip"""
    issue_slip = models.ForeignKey(IssueSlip, on_delete=models.CASCADE, related_name='items')
    item = models.ForeignKey(Item, on_delete=models.PROTECT)
    quantity_requested = models.IntegerField()
    quantity_issued = models.IntegerField(default=0)
```

---

## Integration Functions

### Get Indenter's Department
```python
def get_indenter_department(user):
    """Get department of the indenter"""
    from applications.globals.models import ExtraInfo
    
    extrainfo = ExtraInfo.objects.get(user=user)
    return extrainfo.department
```

### Get HOD for Indent Approval
```python
def get_hod_for_indent(indent):
    """Get HOD for indent approval"""
    from applications.globals.models import HoldsDesignation
    
    department = indent.department
    
    # Find HOD designation for this department
    hod_designations = HoldsDesignation.objects.filter(
        designation__name__icontains='hod'
    ).select_related('working', 'designation')
    
    for hd in hod_designations:
        if department.name.lower() in hd.designation.name.lower():
            return hd.working
    
    return None
```

### Link to Project Budget (RSPC)
```python
def verify_project_budget(project_number, amount):
    """Verify budget availability for project-linked indent"""
    # This would link to RSPC SponsoredProject model
    # Check if sufficient budget is available
    pass
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/ps/api/vendors/` | GET, POST | Vendor management |
| `/ps/api/items/` | GET, POST | Item master |
| `/ps/api/indents/` | GET, POST | Indents |
| `/ps/api/indents/<id>/approve/` | POST | Approve indent |
| `/ps/api/purchase-orders/` | GET, POST | Purchase orders |
| `/ps/api/grn/` | GET, POST | Goods receipts |
| `/ps/api/stock/` | GET | Current stock |
| `/ps/api/issue-slips/` | GET, POST | Issue slips |

---

## Important Sync Points

1. **User Department**: Indent department from `ExtraInfo.department`.
2. **HOD Approval**: Use `HoldsDesignation` to identify HOD.
3. **Project Budget**: Link to RSPC for project-linked purchases.
4. **File Tracking**: Integrate with FTS for indent routing.

---

