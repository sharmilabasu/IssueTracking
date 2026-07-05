# Issue Ingest Windows Service — Specification

## Overview

- Purpose: Daily import of property issues from CSV files in `C:\inbound` into SQL Server.
- Platform: .NET 8 Worker Service (Windows Service) + SQL Server.
- Authentication: Integrated Windows Authentication. Service runs under a domain service account with minimum DB/file permissions.

## Assumptions (confirmed)

- CSVs live in `C:\inbound` (support `issues_YYYYMMDD.csv` pattern). Header row present.
- Upsert behavior: by `IssueID` (update existing, insert new).
- Date formats: common variants accepted; parser tries ISO, `MM/dd/yyyy`, `yyyy-MM-dd`, with time if present.
- Post-processing: success -> `C:\inbound\archive\YYYYMMDD\`; failures -> `C:\inbound\quarantine\`.
- Retention: archive retention default 90 days (configurable).

## Deliverables

- SQL DDL for schema and supporting objects.
- .NET Worker Service design and sample code snippets.
- CSV parsing & mapping rules.
- Error handling & retry strategy.
- Security & permissions checklist.
- Deployment & installation steps (PowerShell + sc.exe examples).
- Test plan and sample data.
- README with run/test steps.

---

**Database Schema (SQL Server DDL)**

CREATE TABLE statements (example types; adjust lengths as needed):

```sql
CREATE TABLE dbo.PropertyIssues (
    IssueID            NVARCHAR(100) NOT NULL PRIMARY KEY,
    UnitNumber         NVARCHAR(50)  NULL,
    Priority           NVARCHAR(50)  NULL,
    Category           NVARCHAR(100) NULL,
    Notes              NVARCHAR(MAX) NULL,
    OpenDate           DATETIME2     NULL,
    ClosedDate         DATETIME2     NULL,
    AssignedTo         NVARCHAR(100) NULL,
    AssignDate         DATETIME2     NULL,
    OpenedBy           NVARCHAR(100) NULL,
    Status             NVARCHAR(50)  NULL,
    TransactionSpecialist NVARCHAR(100) NULL,
    LastModifiedUtc    DATETIME2     NOT NULL DEFAULT SYSUTCDATETIME(),
    LastImportedBatchId INT          NULL
);
GO

CREATE TABLE dbo.ImportBatches (
    ImportBatchId      INT IDENTITY(1,1) PRIMARY KEY,
    FileName           NVARCHAR(260) NOT NULL,
    FilePath           NVARCHAR(500) NOT NULL,
    RowCount           INT NOT NULL,
    ImportedBy         NVARCHAR(200) NULL,
    StartedUtc         DATETIME2 NOT NULL,
    CompletedUtc       DATETIME2 NULL,
    Status             NVARCHAR(50) NOT NULL, -- Pending, Completed, Failed
    ErrorMessage       NVARCHAR(MAX) NULL
);
GO

CREATE TABLE dbo.ImportErrors (
    ErrorId            INT IDENTITY(1,1) PRIMARY KEY,
    ImportBatchId      INT NULL REFERENCES dbo.ImportBatches(ImportBatchId),
    RowNumber          INT NULL,
    RawLine            NVARCHAR(MAX) NULL,
    ErrorMessage       NVARCHAR(MAX) NOT NULL,
    CreatedUtc         DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
GO

-- Optional lookup for normalized transaction specialists
CREATE TABLE dbo.TransactionSpecialists (
    SpecialistId       INT IDENTITY(1,1) PRIMARY KEY,
    Name               NVARCHAR(200) UNIQUE NOT NULL
);
GO
```

Suggested index for frequent queries:

```sql
CREATE INDEX IX_PropertyIssues_Status_Priority ON dbo.PropertyIssues(Status, Priority);
```

Recommended upsert stored proc (high-level):

- `usp_UpsertPropertyIssue(@IssueID, @UnitNumber, ..., @BatchId)` uses MERGE to insert/update and set `LastModifiedUtc`.

---

**Windows Service Architecture**

- Implementation: .NET 8 Worker Service with `Microsoft.Extensions.Hosting.WindowsServices`.
- Components:
  - FileWatcher/Scanner: runs on schedule (daily timer) or watches folder for new files.
  - ImportService: orchestrates parsing, validation, DB upsert, and post-processing.
  - CsvParser: robust parser using `CsvHelper` (nuget) with culture-invariant settings.
  - Retry/Aggregation Engine: handles transient errors with exponential backoff.
  - Logging: `Microsoft.Extensions.Logging` to file (Serilog with rolling file sink) and Windows Event Log.
  - Configuration: `appsettings.json` (overrides via environment or secrets), includes inbound path, archive/quarantine paths, DB connection string (using Integrated Security), schedule, SMTP settings.
  - Health/Telemetry: optional endpoint or event logs for monitoring.

Service behavior:

- On scheduled run:
  - Scan `C:\inbound` for files matching pattern.
  - For each file: create ImportBatch record (Status=Pending), stream-parse CSV row-by-row, validate and upsert with transaction batching (e.g., bulk size 500).
  - On successful batch: move file to archive folder; mark ImportBatch CompletedUtc.
  - On row-level errors: insert ImportErrors and continue (unless error threshold reached).
  - On fatal job failure: mark ImportBatch Failed, move file to quarantine, write event log and optionally send alert.

Configuration example (appsettings.json excerpt):

```json
{
  "Inbound": {
    "FolderPath": "C:\\inbound",
    "ArchivePath": "C:\\inbound\\archive",
    "QuarantinePath": "C:\\inbound\\quarantine",
    "FilePattern": "issues_*.csv"
  },
  "Import": {
    "BatchSize": 500,
    "MaxRowErrorsPerFile": 100
  },
  "Schedule": {
    "DailyRunTime": "02:00"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=SQLSERVER01;Database=IssueTracking;Integrated Security=true;TrustServerCertificate=false;"
  },
  "Logging": {}
}
```

---

**CSV Parsing & Mapping Rules**

- Expect header row. Case-insensitive mapping to fields:
  - Issue ID -> IssueID (required)
  - Unit # -> UnitNumber
  - Priority -> Priority
  - Category -> Category
  - Notes -> Notes
  - Open Date -> OpenDate
  - Closed Date -> ClosedDate
  - Assign To -> AssignedTo
  - Assign Date -> AssignDate
  - Opened By -> OpenedBy
  - Status -> Status
  - Transaction Specialist -> TransactionSpecialist
- Whitespace trimmed, nulls mapped from empty strings.
- Date parsing: try `DateTime.TryParseExact` with list: ISO, `MM/dd/yyyy`, `M/d/yyyy`, `yyyy-MM-dd`, with and without times; fallback to `DateTime.TryParse` with `CultureInfo.InvariantCulture`.
- Validation:
  - `IssueID` presence required — row moved to ImportErrors if missing.
  - If `OpenDate` parse fails -> row error.
  - Priority/Status normalized against configured sets; unknown values logged but accepted as-is unless business rule forbids.
- Upsert logic:
  - Use MERGE or stored proc per row or batch using table-valued parameter (TVP) for efficiency.
  - Set `LastModifiedUtc = SYSUTCDATETIME()` and `LastImportedBatchId`.

---

**Error Handling & Retry Strategy**

- Transient DB errors: retry up to 3 attempts with exponential backoff (1m, 5m, 15m).
- File access errors: retry with backoff; if file locked beyond threshold, move to quarantine and log.
- Row validation errors: insert into `ImportErrors` with raw line and message; do not abort whole file unless row error count exceeds `MaxRowErrorsPerFile`.
- Job failure notifications:
  - Log to rolling files and Windows Event Log.
  - Optional email alert for job-level failure using SMTP config.
- Monitoring: expose success/failure events to Event Log; optional export of ImportBatches for dashboarding.

---

**Security & Permissions (Windows Auth)**

- Service account:
  - Create a domain service account (e.g., DOMAIN\svc-issue-import).
  - Grant the account:
    - File system: read on `C:\inbound`; write on `C:\inbound\archive` and `C:\inbound\quarantine`.
    - SQL Server: create a contained DB user mapped to the service account (Integrated). Grant minimal rights: `db_datawriter`, `db_datareader`, and `EXECUTE` on upsert stored procedures. Avoid sysadmin.
- Network shares:
  - If using UNC path, grant the service account share + NTFS permissions.
- Connection string:
  - Use `Integrated Security=true`. Do not store SQL credentials.
- Secrets:
  - If any secrets needed (SMTP password), protect them using Windows Credential Manager or Azure Key Vault for production.
- TLS:
  - Use encrypted SQL connections (require `Encrypt=True;TrustServerCertificate=False`), ensure certificates are valid.

---

**Deployment & Installation**

- Build and publish:
  - `dotnet publish -c Release -r win-x64 --self-contained false`
- Install as Windows service (PowerShell example):

```powershell
New-Service -Name "IssueImportService" -BinaryPathName "C:\apps\IssueImport\IssueImportService.exe" -Credential (Get-Credential "DOMAIN\svc-issue-import") -DisplayName "Issue Import Service" -StartupType Automatic
```

- Or use `sc.exe`:

```powershell
sc create "IssueImportService" binPath= "C:\apps\IssueImport\IssueImportService.exe" obj= "DOMAIN\svc-issue-import" password= "<password-if-needed>"
```

- Set service logon to domain account and grant "Log on as a service" right to the account.
- Create inbound/archive/quarantine folders and assign permissions.
- SQL: run the DDL scripts to create schema and stored procedures; grant DB permissions to service account.
- Configure `appsettings.Production.json` with production paths and connection string.
- Start service and verify ImportBatches / Event Log entries.

---

**Test Plan & Sample Data**

- Unit tests:
  - CsvParser tests for valid rows, missing fields, date variations, trimming, and malformed lines.
  - Upsert logic tests (in-memory or test DB) for insert/update behavior.
- Integration tests:
  - End-to-end import of a sample file into a test DB; verify `ImportBatches`, `PropertyIssues`, and `ImportErrors`.
- Manual smoke test:
  - Place `issues_TEST.csv` in `C:\inbound` and trigger service run; verify file moved to archive and DB records created.
- Sample CSV (header + one row):

```
Issue ID,Unit #,Priority,Category,Notes,Open Date,Closed Date,Assign To,Assign Date,Opened By,Status,Transaction Specialist
ISS-1001,101,High,Plumbing,"Leaky faucet",2026-06-30,,"John Doe",2026-07-01,"Alice",Open,"Bob Smith"
```

- Test cases:
  - Missing IssueID -> error entry & quarantined row.
  - Date in `MM/dd/yyyy` and `yyyy-MM-dd` -> parse success.
  - Duplicate IssueID with changed notes -> update test.
  - Large file performance test (10k rows).

---

**Deliverables & README Outline**

- Files:
  - SQL scripts: `schema.sql`, `sp_upsert_issue.sql`, `init_indexes.sql`
  - Service source: `src/IssueImportService/*`
  - Config templates: `appsettings.json`, `appsettings.Production.json`
  - Deployment scripts: `install.ps1`, `uninstall.ps1`
  - README.md with run/test steps, sample CSV, and troubleshooting.
- README quick run:

```powershell
# apply DB schema
sqlcmd -S SQLSERVER01 -d IssueTracking -i schema.sql

# publish and install service (example)
dotnet publish -c Release -r win-x64
.\deploy\install.ps1 -ServicePath "C:\apps\IssueImport\IssueImportService.exe" -Account "DOMAIN\svc-issue-import"
```

---

If you want, I can now:

- Generate the full `schema.sql` and `sp_upsert_issue.sql` files, or
- Scaffold the .NET Worker Service project with the main classes and `CsvHelper` integration, or
- Produce the README and deployment scripts.

Which of these would you like me to create next?
