# Report Module Architecture

## Overview

The report module provides two types of functionality:
1. **Dashboard Reports** - Data displayed in HR/Admin dashboard views
2. **File Exports** - Excel/PDF downloads of employee reports

---

## Architecture Diagram

```mermaid
flowchart TD
    subgraph UI["UI Layer (Templates)"]
        HR_REPORTS["hr/reports.html"]
        HR_DASH["hr/dashboard.html"]
        ADMIN_REPORTS["admin/reports.html"]
        ADMIN_DASH["admin/dashboard.html"]
    end

    subgraph CTRL["Controller Layer"]
        HR_CTRL["HrController.java"]
        ADMIN_CTRL["AdminController.java"]
        REPORT_CTRL["ReportController.java"]
    end

    subgraph SVC["Service Layer"]
        REPORT_SVC["ReportService (interface)"]
        REPORT_IMPL["ReportServiceImpl"]
    end

    subgraph EXPORT["Exporter Layer"]
        EXCEL_EXP["EmployeeReportExcelExporter"]
        PDF_EXP["EmployeeReportPdfExporter"]
    end

    subgraph DTO["DTOs"]
        EMP_DTO["EmployeeReportDto"]
        PREM_DTO["PremiumReportDto"]
        CLAIM_DTO["ClaimReportDto"]
        ENR_DTO["EnrollmentReportDto<br/>(COMMENTED OUT)"]
    end

    subgraph DB["Database"]
        EM["EntityManager<br/>(JPQL Queries)"]
    end

    %% Dashboard Data Flow
    HR_REPORTS --> HR_CTRL
    HR_CTRL --> REPORT_SVC
    ADMIN_REPORTS --> ADMIN_CTRL
    ADMIN_CTRL --> REPORT_SVC

    %% Export Flow
    HR_REPORTS -->|"/reports/employees/excel"| REPORT_CTRL
    HR_REPORTS -->|"/reports/employees/pdf"| REPORT_CTRL
    REPORT_CTRL --> REPORT_SVC

    %% Service Implementation
    REPORT_SVC --> REPORT_IMPL
    REPORT_IMPL --> EM
    REPORT_IMPL --> EXCEL_EXP
    REPORT_IMPL --> PDF_EXP

    %% DTOs
    REPORT_IMPL --> EMP_DTO
    REPORT_IMPL --> PREM_DTO
    REPORT_IMPL --> CLAIM_DTO
    EXCEL_EXP --> EMP_DTO
    PDF_EXP --> EMP_DTO
```

---

## Detailed Flow: Dashboard Reports

### HR Dashboard (`/hr/reports`)

```mermaid
sequenceDiagram
    participant U as User (HR)
    participant T as hr/reports.html
    participant C as HrController
    participant S as ReportServiceImpl
    participant DB as Database

    U->>T: Navigate to /hr/reports
    T->>C: GET /hr/reports
    C->>S: getEmployeeCountByOrganization(orgId)
    S->>DB: JPQL Query (Organization + Employee)
    DB-->>S: Result rows
    S-->>C: List<EmployeeReportDto>
    C->>S: getPremiumCollectedByOrganization(orgId)
    S->>DB: JPQL Query (Organization + Enrollment)
    DB-->>S: Result rows
    S-->>C: List<PremiumReportDto>
    C->>S: getClaimSummaryByEnrollment()
    S->>DB: JPQL Query (Enrollment + Claim)
    DB-->>S: Result rows
    S-->>C: List<ClaimReportDto>
    C-->>T: Model with all report data
    T-->>U: Rendered reports page
```

---

## Detailed Flow: File Export

### Excel/PDF Download (`/reports/employees/excel` or `/reports/employees/pdf`)

```mermaid
sequenceDiagram
    participant U as User
    participant T as hr/reports.html
    participant C as ReportController
    participant S as ReportServiceImpl
    participant E as Exporter
    participant DB as Database

    U->>T: Click "Download Excel" button
    T->>C: GET /reports/employees/excel
    C->>S: exportEmployeeReportExcel()
    S->>S: getEmployeeCountByOrganization(null)
    S->>DB: JPQL Query
    DB-->>S: List<EmployeeReportDto>
    S->>E: EmployeeReportExcelExporter.export(data)
    E-->>S: byte[] (Excel file bytes)
    S-->>C: byte[]
    C-->>U: ResponseEntity + ByteArrayResource<br/>(File download)
```

---

## Component Details

### Controllers Using ReportService

| Controller | Endpoint | Methods Used | Purpose |
|------------|----------|--------------|---------|
| [HrController](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/hr/HrController.java#L233-246) | `/hr/reports` | `getEmployeeCountByOrganization`, `getPremiumCollectedByOrganization`, `getClaimSummaryByEnrollment` | HR dashboard reports |
| [AdminController](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/admin/AdminController.java#L123-146) | `/admin/reports` | Same as above | Admin dashboard reports |
| [ReportController](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/report/controller/ReportController.java) | `/reports/employees/excel`, `/reports/employees/pdf` | `exportEmployeeReportExcel`, `exportEmployeeReportPdf` | File downloads |

---

### ReportService Methods

| Method | Return Type | Used By | Status |
|--------|-------------|---------|--------|
| `getEmployeeCountByOrganization(orgId)` | `List<EmployeeReportDto>` | HrController, AdminController | ✅ ACTIVE |
| `getPremiumCollectedByOrganization(orgId)` | `List<PremiumReportDto>` | HrController, AdminController | ✅ ACTIVE |
| `getClaimSummaryByEnrollment()` | `List<ClaimReportDto>` | HrController, AdminController | ✅ ACTIVE |
| `exportEmployeeReportExcel()` | `byte[]` | ReportController | ✅ ACTIVE |
| `exportEmployeeReportPdf()` | `byte[]` | ReportController | ✅ ACTIVE |
| ~~`getEnrollmentCountByPolicy()`~~ | ~~`List<EnrollmentReportDto>`~~ | None | ❌ COMMENTED OUT |

---

### DTOs

| DTO | Fields | Used In |
|-----|--------|---------|
| [EmployeeReportDto](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/report/dto/EmployeeReportDto.java) | `organizationId`, `organizationName`, `employeeCount` | Dashboard + Export |
| [PremiumReportDto](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/report/dto/PremiumReportDto.java) | `organizationId`, `organizationName`, `totalPremiumCollected` | Dashboard only |
| [ClaimReportDto](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/report/dto/ClaimReportDto.java) | `enrollmentId`, `totalClaims`, `totalApprovedAmount` | Dashboard only |
| [EnrollmentReportDto](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/report/dto/EnrollmentReportDto.java) | `policyId`, `policyName`, `enrollmentCount` | ❌ COMMENTED OUT |

---

### Exporters

| Exporter | Output Format | Library Used |
|----------|---------------|--------------|
| [EmployeeReportExcelExporter](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/report/exporter/EmployeeReportExcelExporter.java) | `.xlsx` | Apache POI |
| [EmployeeReportPdfExporter](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/report/exporter/EmployeeReportPdfExporter.java) | `.pdf` | iText |

---

## File Structure

```
report/
├── controller/
│   └── ReportController.java      # Excel/PDF download endpoints
├── dto/
│   ├── EmployeeReportDto.java     # ✅ Active
│   ├── PremiumReportDto.java      # ✅ Active
│   ├── ClaimReportDto.java        # ✅ Active
│   └── EnrollmentReportDto.java   # ❌ Commented out
├── exporter/
│   ├── EmployeeReportExcelExporter.java  # ✅ Active
│   └── EmployeeReportPdfExporter.java    # ✅ Active
└── service/
    ├── ReportService.java         # Interface
    └── ReportServiceImpl.java     # Implementation
```

---

## UI Integration Points

### hr/reports.html (lines 99-105)

```html
<!-- Excel Download -->
<a th:href="@{/reports/employees/excel}" class="btn btn-success btn-sm">
    Download Excel
</a>

<!-- PDF Download -->
<a th:href="@{/reports/employees/pdf}" class="btn btn-danger btn-sm">
    Download PDF
</a>
```

These buttons trigger file downloads via [ReportController](file:///c:/Users/lucky/.antigravity/employee-insurance-management-master/src/main/java/com/employeeinsurancemanagement/report/controller/ReportController.java).
