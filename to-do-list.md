# Implementation To-do List

Updated: 2026-07-03

## Phase 1 — Core ingestion

- [ ] Define and create SQL schema for issue import
  - Acceptance criteria:
    - `PropertyIssues`, `ImportBatches`, and `ImportErrors` tables are defined in a SQL script.
    - Primary keys, foreign keys, and at least one supporting index are included.
    - Schema script can be executed without syntax errors in SQL Server.

- [ ] Build CSV parsing logic for the inbound file format
  - Acceptance criteria:
    - The parser reads `C:\inbound` files matching `issues_*.csv`.
    - It accepts header row mappings for the required columns.
    - It trims whitespace and converts empty values to null.

- [ ] Map CSV columns to `PropertyIssues` fields
  - Acceptance criteria:
    - Each CSV header is mapped correctly to the DB model.
    - Required field `IssueID` is validated.
    - Date values are parsed from common formats like `yyyy-MM-dd` and `MM/dd/yyyy`.

## Phase 2 — Import processing

- [ ] Implement import batch tracking
  - Acceptance criteria:
    - `ImportBatches` records are created for each file import attempt.
    - Batch status can be `Pending`, `Completed`, or `Failed`.
    - Start and completion timestamps are recorded.

- [ ] Add upsert persistence for issue rows
  - Acceptance criteria:
    - CSV rows are inserted or updated in `PropertyIssues` by `IssueID`.
    - `LastModifiedUtc` is updated on each import.
    - Duplicate issue imports update the existing record.

- [ ] Add row-level error handling and logging
  - Acceptance criteria:
    - Invalid rows are recorded in `ImportErrors` with row number and message.
    - The service continues processing remaining rows when row-level errors occur.
    - Files with row-level failures are not silently ignored.

## Phase 3 — Reliability and operations

- [ ] Add file archive and quarantine behavior
  - Acceptance criteria:
    - Successfully imported files are moved to `C:\inbound\archive\YYYYMMDD\`.
    - Failed files are moved to `C:\inbound\quarantine\`.
    - File moves are atomic and log failures when file operations fail.

- [ ] Implement retries for transient failures
  - Acceptance criteria:
    - Transient database or file access errors retry automatically.
    - Retry policy uses exponential backoff.
    - Persistent failures after retries mark the batch as `Failed`.

- [ ] Add service logging and Windows Event Log integration
  - Acceptance criteria:
    - The worker logs import progress and errors to a rolling file.
    - Critical failures also write to the Windows Event Log.
    - Logs include batch ID, file name, row counts, and error details.

## Phase 4 — Security, deployment, and testing

- [ ] Configure Integrated Windows Authentication and service account permissions
  - Acceptance criteria:
    - The service configuration uses `Integrated Security=true` for SQL Server.
    - The domain service account can read `C:\inbound` and write archives/quarantine.
    - The account has minimal SQL permissions to read/write the import tables.

- [ ] Create service deployment and installation scripts
  - Acceptance criteria:
    - Publish instructions are documented in Markdown.
    - PowerShell or `sc.exe` install/uninstall scripts are available.
    - The service can be installed and started on Windows.

- [ ] Define test cases and sample data for verification
  - Acceptance criteria:
    - Sample CSV files exist for valid import, missing `IssueID`, duplicate `IssueID`, and invalid date formats.
    - A test plan lists unit and integration verification steps.
    - Import result validation checks `PropertyIssues`, `ImportBatches`, and `ImportErrors`.
