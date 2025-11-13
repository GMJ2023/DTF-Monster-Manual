# 🧩 Data Transformation & PDF Processing System 
###   The Monster Manual

**Author:** Geoffrey Jones

*“The drop zone for all your spreadsheets and PDFs”*  


*This document explains how the Data Transformation Framework (DTF) automates the entire lifecycle of file handling and transformation, from ingestion to upload.*
---

## 1. Overview

**Purpose:**  
Provide an end-to-end description of the automated data transformation and PDF processing system, detailing how files flow through RPA, Catalyst, PowerShell and external services to produce master-ready CSV outputs for the online payroll application.

*An end-to-end guide to the Data Transformation Framework powering Zoho RPA, Catalyst and beyond.*

**Core Functions:**  
- Automated ingestion of agency data and mapping templates  
- CSV transformation and upload  
- PDF parsing, data extraction and output to CSV Paradise  
- Periodic dispatch to the online payroll application

**Tech Stack Summary:**  
- **Zoho RPA:** Orchestration and flow control  
- **Zoho Catalyst:** Serverless functions for processing and integration  
- **PowerShell:** Local execution and script automation  
- **AWS S3:** Intermediate storage and presigned URL generation  
- **OneDrive:** Input file sources  
- **Online payroll App:** Final data destination  

---

## 2. Architecture Diagram (Overview)

*(Insert simple flow diagram later — placeholder below)*

```
Agency CSVs / PDFs
      ↓
OneDrive → S3 (Upload via RPA)
      ↓
Catalyst Functions (Transform / Parse)
      ↓
CSV Paradise → online payroll application flow → Archive
```

---

## 3. Components

### 🧠 Zoho RPA
- **Purpose:** Automates file detection, upload and invocation of Catalyst functions.  
- **Key Blocks:**
  - Open Application – triggers PowerShell scripts  
  - Wait / Delay – controls sequencing  
  - Decision Blocks – file existence and retries  
- **Known Behaviours:**  
  - Avoid duplicate triggers (ensure lock file active)  
  - Desktop event watchers for new PDF arrivals  

---

### ☁️ Zoho Catalyst Functions
| Function Name | Type | Purpose | Key Notes |
|----------------|------|----------|-----------|
| `dataTransformer` | Basic I/O | Converts agency CSVs into master format | Reads mapping CSV, applies transformations |
| `updateAgencyImport` | Advanced I/O | Fetches and overwrites static OneDrive files | Uses OAuth-managed connection |
| `resolveAgencyName_node` | Advanced I/O | Matches agency names via CRM | Uses connection `crmagencyv1` |
| `pdfPool_node` | Advanced I/O | Extracts structured data from agency PDFs | Includes agency-specific parsers and CSV output |

---

### 🖥️ PowerShell Scripts
| Script Name | Purpose | Notes |
|--------------|----------|-------|
| `upload_agency_and_template.bat` | Uploads CSVs to S3 and writes presigned URLs to `transform_args.txt` | Triggered via RPA |
| `run_transform.bat` | Invokes Catalyst transformation with URLs from `transform_args.txt` | |
| `write_text_and_invoke.ps1` | Receives PDF text, writes `.txt`, invokes parser | |
| `run_pdf_parser_wrapper.ps1` | Manages locking, invokes inner script safely | Includes `.lock` file logic |

---

### 🪣 AWS S3
- **Bucket:** `agencyimport`
- **Region:** `eu-north-1`
- **Usage:** Temporary file hosting for transformation and PDF parsing  
- **Notes:**  
  - Pre-signed URLs expire; ensure upload script runs before invocation  
  - Bucket structure kept flat for simplicity  

---

### 📁 Local & Shared OneDrive Folders

| Folder Name       | Description                                                  | Function in Workflow                                         |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Timesheets**    | Primary inbound folder monitored by Zoho RPA desktop events. | Receives incoming CSV, XLSX and PDF files. Each new arrival triggers the start of the DTF pipeline. |
| **Holding Zone**  | Temporary buffer used to prevent file conflicts when a new file shares the same name as one still being processed upstream. | Holds duplicate or queued files safely until the existing process finishes, ensuring the correct sequence of operations. |
| **CSV Paradise**  | Central drop zone for parsed CSV files produced by the PDF Pool. | Stores accumulated CSVs before timed dispatch to the Drop Zone. |
| **CSV Drop Zone** | Receives CSVs moved by the scheduled mover service.          | Acts as a buffer before file conversion and Flow hand-off.   |
| **AssembleXLSX**  | Repository for converted XLSX files ready for transmission, or a safe holding point if Zoho RPA is not enabled. | Waits for the Zoho RPA Flow to resume before forwarding data to the Formatted folder. |
| **Formatted**     | Final staging area used by the payroll automation Flow.      | Files delivered here are validated and passed to the payroll application or redirected in maintenance mode. |



**Workflow Summary:**
PDFs and CSVs arrive in the **Timesheets** folder. They are validated and staged in the **Holding Zone** when necessary — typically when a file with the same name is already being processed further up the pipeline. Once clear, files progress through the DTF’s transformation services.

Parsed pdf outputs accumulate in **CSV Paradise**, are moved into the **CSV Drop Zone**, converted to XLSX within **AssembleXLSX**, and finally dispatched via **Zoho Flow** to the **Formatted** folder.

If the **Zoho RPA Flow** is temporarily disabled or paused, the system safely holds files within **AssembleXLSX** until the flow is re-enabled, ensuring no data are lost or sent prematurely.

The Flow itself acts as a *maintenance-mode switch*, allowing the process to continue safely during testing or updates without transmitting data to the live payroll system.

---

## 4. Service Backbone

### 🧩 File Monitor Service (`file_monitor_service.ps1`)
**Purpose:**  
Continuously monitors the `Timesheets` folder for newly dropped files, validates their type and triggers the correct downstream process (transformation, parsing or rejection). This script acts as the event controller for the whole automation flow.

**Context:**  
Scheduled as a background service and runs independently of Zoho RPA, ensuring resilience even if RPA restarts or fails mid-process.

**Key Functions:**
- **File Detection:** Watches the `Timesheets` directory for new files using `System.IO.FileSystemWatcher`.
- **File Validation:**  
  - Accepts only `.csv`, `.xlsx`, `.xls` and `.pdf`.  
  - Rejects unsupported extensions gracefully with alerts.
- **Routing Logic:**  
  - CSV and Excel files are queued for transformation.  
  - PDFs are passed to the parsing pipeline.  
  - Invalid files are logged and moved to a Rejected folder.  
- **Logging:** Every event (new file, rejection, movement) is timestamped in the service log for traceability.

**Integration Points:**
- Triggers subsequent scripts such as `move_holding_to_assemble.ps1` or PDF-specific handlers depending on file type.
- Communicates status updates to Zoho RPA or local notification systems.
- Forms the first stage in the automation lifecycle.

**Design Justification:**  
Switching from WorkDrive to local monitoring solved file corruption issues caused by encoding and BOM changes. The local model preserves full control over file integrity and timing.

**Resilience Features:**
- Retries if files are locked or partially written.  
- Graceful error handling with timestamped exceptions.  
- Runs indefinitely as a background task or Windows Scheduled Task.  

> This service ensures every incoming file is handled exactly once, correctly classified and safely handed over to the next process — forming the backbone of the data transformation pipeline.

---

### 📦 CSV Drop Zone Mover (`move_csvs_to_dropzone.ps1`)

Moves final CSV outputs from **CSV Paradise** to the **CSV Drop Zone** on a timed schedule.  
PDF-parsed files are first written into **CSV Paradise**, where they accumulate until the next dispatch cycle.  

This allows multiple PDF reads to be combined into a single CSV before dispatch.  
If the scheduler triggers while PDFs are still being processed, a new CSV file is created and the remaining data continues to accumulate. The next scheduled run will move those files automatically.  

This approach prevents the system from generating one CSV per member, avoiding unnecessary file floods in the online payroll application.

---

### 🚀 Browser Initialiser and Watchdog  
(`Start_browser_new.bat` and `monitor_browser.ps1`)  

**Purpose:**  
Together, these two components maintain a reliable, authenticated browser session for Zoho RPA automation. The system uses a blend of manual control and automated monitoring to keep the environment stable while respecting Captcha and login restrictions.  

- The **Browser Initialiser** (`Start_browser_new.bat`) is a **manual launcher** used by the user to start the browser session and complete the Captcha challenge.  
- The **Browser Watchdog** (`monitor_browser.ps1`) runs automatically as a **Windows Scheduled Task**, checking that the session remains open and responsive.  

**Context:**  
Some RPA tasks depend on a persistent browser session to upload files or trigger Catalyst functions. Because Captcha authentication cannot be bypassed, this controlled approach combines user-initiated login with automated oversight.  

**Workflow:**  
1. The user manually runs `Start_browser_new.bat` to launch the browser using the correct profile and complete any Captcha verification.  
2. Once authenticated, the browser session is established and its DevTools WebSocket endpoint recorded in `curl_endpoint_final.txt`.  
3. The **Browser Watchdog** periodically checks that the browser is still running and connected.  
4. If the browser is closed or becomes unresponsive, the watchdog sends a friendly alert to the user, prompting them to restart it manually via the `.bat` script.  

**Alert Example:**  
When the browser is detected as closed, the watchdog sends the following message to the operations team:  

```
Hi Team,

Browser Failure: Chrome browser with --remote-debugging-port=9222 not running

Go ahead and restart Chrome by double-clicking Start_browser_new.bat on the VM desktop.
When it opens, just log in to the payroll web portal.

Timestamp: 21-10-2025 15:16:25
```

**Key Functions:**  
- **Initialiser:**  
  - Opens the automation browser with the correct profile.  
  - Establishes and records the WebSocket endpoint for RPA integration.  
  - Keeps the session open for real-time monitoring and task execution.  
- **Watchdog:**  
  - Runs on a timed schedule to confirm the browser’s presence and responsiveness.  
  - Detects unexpected closure or session loss.  
  - Sends a clear, polite alert to the user with restart instructions.  
  - Logs all events and checks with timestamps for audit and diagnostics.  

**Design Justification:**  
This hybrid design balances security and reliability. Full automation is restricted by Captcha enforcement, so user-driven startup ensures compliance while the watchdog maintains operational awareness and prevents unnoticed downtime.  

**Resilience Features:**  
- Friendly, non-intrusive alerts for user action.  
- Continuous background checks without performance impact.  
- Timestamped logging for transparency and troubleshooting.  

**Future Outlook:**  
Once the payroll system provides secure API endpoints for login and file upload, this process can be fully automated — eliminating the manual browser stage entirely.  

> These two scripts work in harmony — one waking the Data Transformation Framework (DTF), the other keeping watch while it roams.

---

## 5. Data Flow / File Lifecycle

1. **File Arrival:**  
   Agency CSV or PDF dropped into monitored folder.  
2. **Upload Phase:**  
   RPA runs upload script → sends file(s) to S3 → generates URLs.  
3. **Transformation / Parsing:**  
   Catalyst function invoked with URLs → processes → outputs CSV.  
4. **Post-Processing:**  
   PDF parsed to CSV format are written to CSV Paradise → periodically moved to Drop Zone.  
5. **Dispatch:**  
   Online payroll application flow ingests XLSX → completes automation loop.  

---

## 6. Error Handling & Logging  

**Purpose:**  
Provide a unified approach for tracking, diagnosing and recovering from any issues across RPA, Catalyst and PowerShell layers. Every component logs its own activity locally or in the cloud, creating a clear audit trail that can be used to trace failures or confirm successful runs.  

**Unified Logging:** Each parser logs to the Catalyst console and the local log stream, providing full traceability across a centralised monitoring environment.

---

### 🧩 Multi-Layered Logging Architecture  

| Layer | Source | Log Location | Description |
|-------|---------|---------------|--------------|
| **RPA** | Zoho RPA Flow | RPA Execution Logs (Zoho Console) | Captures block outcomes, delays, and exceptions. |
| **Catalyst** | Catalyst Functions | Catalyst Console Logs | All Deluge/Node/Java log output is timestamped and visible via the console. |
| **PowerShell** | Local Scripts | `C:\binary-proxy\logs\` | Each PowerShell service (monitor, parser, mover) writes its own `.log` file with timestamps. |
| **Browser Monitor** | `monitor_browser.ps1` | `C:\binary-proxy\logs\browser_watchdog.log` | Records process checks, failures and alerts issued to users. |

---

### ⚙️ Logging Format  

Each PowerShell log entry follows a consistent structure for easy searching and pattern matching:  

```
[2025-10-22 14:38:07] [INFO]  File detected: Drivers_Timesheet_2210.csv
[2025-10-22 14:38:09] [WARN]  File locked, retrying in 15 seconds
[2025-10-22 14:38:32] [SUCCESS]  File moved to Assemble
[2025-10-22 14:38:33] [ERROR]  Failed to connect to Catalyst endpoint: Timeout
```

---

### 🛠️ Recovery and Restart Strategy  

- **File Locks:**  
  If a file is still being written, the monitor retries after a short delay. Persistent locks are logged as warnings, not errors.  

- **Stale Lock Files:**  
  The global parser lock file (`$env:TEMP\global_pdf_parser.lock`) can occasionally remain after a crash or manual termination.  
  To recover safely:  
  ```powershell
  Remove-Item $env:TEMP\global_pdf_parser.lock
  ```
  This releases the lock and allows the next parsing job to continue normally.  

- **Expired URLs:**  
  S3 pre-signed URLs can expire if uploads and transformations are delayed. In that case, re-run the upload script (`upload_agency_and_template.bat`) to generate new links before re-invoking Catalyst.  

- **Network Timeouts:**  
  Catalyst or S3 calls occasionally fail during high traffic. These are automatically retried once before logging an error entry and sending a local alert.  

- **RPA Failures:**  
  Zoho RPA includes built-in retry blocks and error branches for handling intermittent execution issues. Logs can be reviewed in the RPA console for failed tasks.  

---

### 🔔 Alerting  

Certain scripts (notably `monitor_browser.ps1` and the File Monitor Service) send notifications when manual action is required — for example, browser restarts, invalid file types or repeated lock failures.  
All alerts follow the same polite, human-readable style for clarity and consistency.  

---

### 🧠 Design Philosophy  

Error handling is built on three guiding principles:  

1. **Detect Early:** Always log before failure becomes silent.  
2. **Recover Gracefully:** Retry where possible, and pause rather than crash.  
3. **Inform Clearly:** Alert humans in plain English when manual action is needed.  

This layered design ensures that even if a process stops, the Data Transformation Framework (DTF) leaves a full breadcrumb trail explaining what went wrong and how to fix it.  

---

## 7. Known Quirks & Gotchas  

**Purpose:**  
Capture the edge cases, unexpected behaviours and subtle timing issues discovered during live use of the Data Transformation Framework (DTF) system. These are not bugs — they’re characteristics of how the system behaves under real-world conditions.  

---

### 🧩 RPA Execution Timing  
- Zoho RPA can sometimes trigger more than one `.bat` file simultaneously, especially if multiple *Open Application* blocks are configured.  
- **Fix:** Ensure that only one *Open Application* block is connected and that each script creates a `.lock` file while running.  
- **Tip:** Use the wrapper PowerShell script (`run_pdf_parser_wrapper.ps1`) to prevent re-entry or overlap.  

---

### 📂 File Writing & Locks  
- Files arriving from network drives may appear before they are fully written, causing read conflicts.  
- **Fix:** The File Monitor Service retries on lock detection with a short delay.  
- **Tip:** Avoid dropping multiple large files simultaneously to reduce disk contention.  

---

### 📊 CSV Integrity  
- Some CSVs include blank trailing rows or headers with extra commas, which can lead to partial or silent failures during transformation.  
- **Fix:** The transformation function trims and sanitises input automatically, but upstream validation is still recommended.  
- **Tip:** Open suspect files in a plain-text editor to check for invisible delimiters or hidden BOM markers.  

---

### 📄 PDF Parsing (OCR Limitations)  
- PDFs created by optical character recognition occasionally merge words or split float values, leading to parsing inconsistencies.  
- **Fix:** The parsing logic detects common spacing and alignment issues and compensates automatically.  
- **Tip:** Always verify total amounts in the output CSV against the PDF’s summary section.  

---

### 👥 Members Details and Pay Data  
- Detection of values such as *persons details, amounts* and *rates* depends on consistent text order and spacing — in some PDFs, unrelated labels appear close by and must be ignored.  
- **Fix:** The parser skips non-numeric text when identifying key numeric values.  
- **Tip:** Keep a reference sample of each layout to verify the parser’s matching behaviour after updates.  

---

### 🌐 Network & Service Dependencies  
- Catalyst functions may time out during high network load, particularly when processing multiple files in succession.  
- **Fix:** Local retry logic has been added to each function call.  
- **Tip:** Stagger RPA triggers slightly if large batches are being processed together.  

---

### 💬 Logging Volume  
- Verbose logging is useful during development but can produce large `.log` files over time.  
- **Fix:** Logs rotate weekly via Windows Scheduled Task.  
- **Tip:** Archive older logs to a subfolder or cloud store to keep the main log folder tidy.  

---

**Summary:**  
These quirks highlight how resilient the Data Transformation Framework (DTF) has become through practical experience. Each one has been documented, mitigated or automated where possible — proof that the system learns from its environment, just like its creator.  

---

## 8. Long-Term Roadmap 

**Purpose:**  
Outline the planned developments and opportunities for extending the Data Transformation Framework (DTF) system. These are areas identified during live use where additional automation or integration could bring measurable gains in reliability and efficiency.  

---

### 🔄 Full Automation of Browser Operations  
- Replace the manual start of the browser and Captcha step with secure API authentication once the payroll portal provides programmatic access.  
- This will remove the need for `Start_browser_new.bat` and simplify monitoring, allowing headless or container-based execution.  
- Expected benefits: improved uptime, no manual intervention, and cleaner alert handling.  

---

### ☁️ Enhanced Cloud Integration  
- Extend S3 usage beyond temporary file storage to a managed cloud archive for historical CSVs, logs and reports.  
- Consider using lifecycle rules for automated retention and clean-up.  
- Evaluate potential migration to Zoho WorkDrive API or other storage providers when full binary preservation is guaranteed.  

---

### 🧠 Adaptive Parsing Engine  
- Generalise the PDF parsing logic into a plugin-style framework allowing new layouts to be added without code duplication.  
- Introduce pattern libraries for detecting names, totals and pay data, so that new formats can be deployed rapidly.  
- Potential to include AI-based text pattern recognition for OCR-heavy documents.  

---

### 🧰 Unified Configuration and Control Panel  
- Create a lightweight desktop dashboard (PowerShell + HTML or Node.js Electron) to display real-time process status, active locks, queue length and recent alerts.  
- Centralise configuration for directories, intervals and thresholds, reducing manual edits to `.ps1` files.  

---

### 📈 Analytics and Reporting  
- Introduce a reporting pipeline that aggregates daily processing stats (files handled, average run time, errors recovered).  
- Export these metrics to Zoho Analytics or a simple CSV summary for management visibility.  

---

### 🧩 Code Modularisation and Distribution  
- Package each PowerShell service and Catalyst function into self-contained modules with semantic versioning.  
- Enable easier deployment, rollback and external sharing between environments or projects.  

---

**Summary:**  
The Data Transformation Framework (DTF) has proven reliable under pressure. Future development will focus on making it self-sustaining, observable and easily extended — a platform that not only automates the mundane but adapts as new formats and processes emerge.  

---




---

## Technical Appendices

The following appendices provide a detailed technical reference for the key components of the Data Transformation Framework (DTF). Each section explains how its respective layer contributes to the overall automation pipeline — from cloud-based transformation logic to on-premise orchestration and dynamic parsing.

---

## Appendix A – Zoho Catalyst Functions

*Together, these Catalyst functions form the cloud backbone of DTF, translating raw input into structured output while maintaining traceability and audit readiness.*

### 1. `dataTransformer_node`

#### **Overview**
`dataTransformer_node` is the core function within the Data Transformation Framework (DTF). It automates the conversion of uploaded CSV data into a consistent master format based on a mapping file. The function runs entirely on Zoho Catalyst’s serverless infrastructure, allowing scalable, cloud-based transformations triggered by Zoho RPA or external scripts.

The function:
- Fetches two downloadable CSVs (an input dataset and a template mapping).
- Detects which mapping template fits the input file structure.
- Applies transformation rules dynamically to reformat the data.
- Produces a normalised output CSV with a consistent structure.
- Generates metadata and log files for downstream automation.

#### **Inputs and Outputs**

| Type | Name | Description |
|------|------|--------------|
| Input | `runId` | Unique identifier for the transformation run |
| Input | `agencyCsvUrl` | Downloadable URL for the input CSV |
| Input | `mappingCsvUrl` | Downloadable URL for the mapping template |
| Input | `originalFileName` | Original filename used for output naming |
| Output | JSON | Includes status, row count, selected template, and base64 CSV |
| Output | File | `/tmp/transformed_output_<runId>.csv` – formatted output |
| Output | File | `/tmp/proposedname_<runId>.json` – naming metadata |
| Output | File | `/tmp/mapUsed.txt` – record of the template used |

#### **Core Logic Flow**
1. **Input Validation**  
   Reads and validates the JSON request body, confirming required parameters (`runId`, `agencyCsvUrl`, `mappingCsvUrl`). Returns HTTP 400 on missing data.
2. **Parallel CSV Download**  
   Uses `axios` to fetch both the agency and mapping CSV files concurrently via HTTPS.
3. **Template Matching**  
   The `matchTemplate()` function compares headers from the agency file with mappings to identify the best-fit template automatically.
4. **Dynamic Template Logic**  
   Depending on the selected template, different logic branches execute.
5. **Transformation Rules**  
   Each field in the input is transformed according to the `TransformationRule` column of the mapping file, powered by the helper functions from `helpers.js`.
6. **Output and Validation**  
   Data is sorted by surname and firstname for consistency. Duplicate rows are flagged in logs.
7. **Response Delivery**  
   On success, returns JSON with run ID, selected template, processed row count, and inline CSV data.

#### **Failure Handling and Logging**
- Each stage logs to Catalyst’s console and writes timestamped `.log` files in `/tmp` on error.  
- Handles failed downloads, invalid JSON, missing mappings, and write errors gracefully.  
- Returns structured JSON error payloads for any failure.

A scheduled companion script (`monitor_output_log.ps1`) reviews DTF output logs and alerts via Zoho webhook if a transformation completes with zero rows, providing a secondary safeguard against silent failures.

---

### 2. `helpers.js`

#### **Overview**
`helpers.js` provides shared parsing and transformation logic for the DTF’s Catalyst functions, improving maintainability and reuse.

#### **Key Functions**
| Function | Purpose |
|-----------|----------|
| `parseCsvRow(csv, headerRow)` | Extracts and normalises CSV header columns. |
| `applyTransformationRule(rule, value)` | Interprets transformation rules (split, default, passthrough). |
| `transformRows(parsedData, mappings, dynamicOT)` | Executes template-driven row transformation with optional overtime logic. |
| `handler(context, event)` | Catalyst-compatible entry point wrapping logic with structured responses. |

#### **Design Choices**
- **Readable Logging** for easier debugging.  
- **Dynamic Mapping** for flexible file structures.  
- **Default Fallbacks** for invalid rows.  
- **Catalyst Compatibility** with both Basic and Advanced I/O.

---



### 3. `resolveAgencyName_node`

#### **Overview**
`resolveAgencyName_node` is an Advanced I/O Zoho Catalyst function responsible for **fuzzy-matching agency names** from uploaded data against existing Accounts in Zoho CRM.  
It automates agency identification during PDF parsing and CSV transformation, ensuring that each record is linked to the correct CRM account even when input names differ slightly.

The function:
- Retrieves all **Recruitment Agency** records from Zoho CRM using OAuth authentication.  
- Cleans and normalises input text (removing punctuation, case, and suffixes).  
- Applies **string-similarity scoring** to find the best match.  
- Falls back to a **secondary hint** (e.g. account type or filename fragment) when confidence is low.  
- Returns structured JSON with the top ten matches, confidence score, and fallback status.

#### **Inputs and Outputs**

| Type | Name | Description |
|------|------|--------------|
| Input | `accountName` | Raw name extracted from file or text |
| Input | `secondaryHint` | Optional contextual hint (e.g. “Healthcare” or file prefix) |
| Input | `selectedTemplate` | Template name influencing match weighting |
| Output | JSON | Best match record, top 10 matches list, match score, and flags |

#### **Core Logic Flow**
1. **Token Acquisition** – Refreshes a Zoho CRM access token using either environment variables or fallback credentials.  
2. **Data Fetch** – Pulls all “Recruitment Agency” Accounts (`Account_Name`, `Account_ID`, `Email`) from CRM via paginated API calls.  
3. **Cleaning & Preparation** – Normalises both the input and CRM names for accurate comparison.  
4. **Primary Match** – Calculates similarity using the `string-similarity` library.  
5. **Template Boosting** – Slightly raises scores for matches containing the selected template name.  
6. **Fallback Match** – If confidence < 0.41, retries matching with `secondaryHint`.  
7. **Caching & Response** – Stores the last queried account in Catalyst Cache for diagnostics and returns a detailed JSON payload.

#### **Failure Handling & Logging**
- Up to three token-refresh attempts with 2 s back-off.  
- Full error context logged to Catalyst Console (including API responses if available).  
- Responds with HTTP 500 and structured error JSON on failure.  

#### **Dependencies**
- `axios` – HTTP client for Zoho API requests.  
- `string-similarity` – Library for Levenshtein-based matching.  
- `zcatalyst-sdk-node` – Catalyst runtime integration and cache access.  

#### **Design Notes**
- Prioritises clarity and predictability over AI-driven matching.  
- Performs full CRM fetch on each invocation to ensure fresh data (optimisation possible later via CRM search API or cache).  
- Balances speed and accuracy for use within larger PDF and CSV pipelines.

---


## Appendix B – PowerShell Scripts

*See Section 6 – Error Handling for complementary retry and alert logic.*

### 🕵️‍♂️ `monitor_output_log.ps1`

#### **Overview**
`monitor_output_log.ps1` is a scheduled watchdog that checks recent DTF output logs for signs of inactivity or failed transformations. Its primary role is to catch scenarios where a CSV file is successfully generated but contains zero data rows — a silent failure that might otherwise go unnoticed.

When such an event is detected, the script automatically alerts the user through a **Zoho webhook**, prompting immediate attention.

#### **Trigger**
- Runs as a **Windows Scheduled Task** every 15 minutes.  
- Executes silently in the background.  
- Checks the most recent transformation output log in the designated folder (`C:\binary-proxy\logs\`).

#### **Logic Summary**
1. **Locate Latest Output Log**  
   The script identifies the newest `.log` file matching the DTF’s naming convention.  
2. **Extract Record Count**  
   Parses the log for `RowCount=` or similar metrics.  
3. **Detect Zero Activity**  
   If the record count is `0`, the system considers this a potential failure.  
4. **Send Webhook Alert**  
   Builds a JSON payload and posts it to a Zoho Flow webhook endpoint.  
5. **Log Outcome**  
   Appends results to `monitor_output_log_history.txt` for audit purposes.

#### **Resilience Features**
- Graceful failure handling for locked or missing logs.  
- Retries webhook on timeout.  
- Maintains a local counter to prevent repeated alerts.

#### **Design Philosophy**
This script adds a safety net above the Catalyst layer, ensuring the DTF’s success criteria aren’t just “function executed” but “data delivered.”  
By tying directly into Zoho’s alerting infrastructure, it closes the feedback loop — giving users immediate visibility if something looks wrong.

---

## Appendix C – Node.js Modules

*This module ensures that local automation and cloud transformation remain perfectly synchronised.*

### **Overview**
Node.js modules power the PDF parsing and flexible extraction logic within the DTF. These modules handle unstructured data, such as OCR-extracted text, converting it into structured CSV outputs.

### **Key Concepts**
- **Parser Modules:** Dedicated scripts (e.g. `flexParser.js`, `index_4rec.js`) designed per format.  
- **Extraction Logic:** Uses regex and block parsing to identify consistent data regions.  
- **Hybrid Matching:** Combines rule-based and fuzzy logic to adapt to varied file structures.  
- **Integration:** Outputs processed data to CSV Paradise for scheduled dispatch via PowerShell movers.


### `upload_to_s3.js`

#### **Overview**
`upload_to_s3.js` is a Node.js utility that manages the secure upload of processed CSV and mapping files to AWS S3. It acts as the bridge between PowerShell automation, Zoho Catalyst functions, and cloud storage.

#### **Key Features**
- Waits for file write completion before upload  
- Uses AWS SDK v2 to securely transfer to S3  
- Generates a presigned download URL (valid 1 hour)  
- Handles transient read/write errors with retry logic  
- Writes lightweight JSON metadata for audit and diagnostics  

#### **Workflow**
1. Delays briefly to ensure file closure  
2. Reads file with retry logic  
3. Uploads to S3 bucket `agencyimport` (region `eu-north-1`)  
4. Returns a presigned URL for use by Catalyst functions or RPA  

#### **Resilience & Error Handling**
- Retries on `EPERM`, `EBUSY`, or `ENOENT`  
- Catches unhandled promise rejections and exits with clear status  
- Logs `[INFO]`, `[WARN]`, and `[ERROR]` messages for batch tracking  



## 🧠 Appendix D – PDF Pool and Parsing Framework

The **PDF Pool** converts unstructured PDF remittance files into normalised CSV data that flow through the same DTF pipeline as spreadsheet imports. It combines PowerShell orchestration, Zoho Catalyst server logic and Node.js parsers to transform complex payroll documents into a unified data model.

------

### Overview

When a PDF is dropped into the monitored **Timesheets** folder, RPA extracts its text and uploads it to Zoho Catalyst in sequential chunks. The **`pdfPool_node`** function retrieves these pieces, reassembles the document and selects the correct parser based on layout signatures and keywords.

Each parser module is dedicated to a specific document family but follows a common pattern: detect block boundaries, extract key data fields, and output consistent rows for CSV generation.

------

### Core Components

| Module                              | Purpose                                                      | Notes                                                        |
| ----------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **`index.js`**                      | Central controller of the PDF Pool. Retrieves text chunks from Catalyst’s `PdfChunks` table, segments the content into “member blocks” (between `Sheet: TS_###` and `Total For Sheet TS_###`), and calls the appropriate parser. | Generates CSV output and returns a base64-encoded payload to RPA. |
| **`flexParser.js`**                 | Default, highly tolerant parser used when no specific format match is found. Handles OCR noise, glued numeric fields, expense lines and VAT detection. | Defines `parseBlock()` and `flexExtract()` functions.        |
| **`index_4Rec.js`**                 | Parser for one of the original payroll layouts. Implements deterministic line-by-line extraction using consistent headings and spacing. | Produces stable results even with minimal variation.         |
| **`index_TXM.js` / `TXMParser.js`** | Dual module for a structurally strict remittance style. Uses layered regex capture to identify worker IDs, week endings and totals spread across multiple pages. | Ensures numeric accuracy across paginated documents.         |
| **`smart_parser.js`**               | Lightweight development and diagnostic parser. Used to prototype recognition patterns and test fallback rules before production deployment. | A safe playground for parser innovation.                     |

------

### Parsing Sequence

1. **Reassembly** – Retrieve and join all `PdfChunks` in order.
2. **Segmentation** – Identify individual “member sheets” using regular-expression markers.
3. **Extraction** – For each segment, parse:
   - Worker ID, Surname and Forename
   - Week-ending and Payment dates
   - Hours / Units, Rate and Total Paid
   - VAT or Expense values where present
4. **Validation** – Apply fallback logic if expected data are missing, ensuring each row remains complete.
5. **CSV Output** – Write normalised results to **CSV Paradise**, from where the scheduled mover dispatches them to the downstream OneDrive / Zoho Flow pipeline.

------

### Error Handling and Extensibility

- **Graceful Degradation:** Parsers continue to output partial rows with placeholders if data are missing.
- **OCR Tolerance:** Cleans text artefacts such as joined floats or stray symbols before pattern matching.
- **Easy Expansion:** New formats can be supported by cloning an existing parser and registering it within `index.js`.
- **Logging:** Each parser emits warnings for blocks that fail to produce a valid `employeeid`, aiding debugging without halting execution.

------

### Workflow Continuity

All CSVs produced by the PDF Pool flow through the same audited route as spreadsheet data — from **CSV Paradise** to **CSV Drop Zone**, then to **AssembleXLSX** and onward via Zoho Flow to the **Formatted** folder.
 This guarantees unified data lineage, consistent logging and transparent recovery across the DTF ecosystem.

---

## 9. Future Enhancements

### 🔗 Orchestrator Integration  
The next major milestone for the Data Transformation Framework (DTF) is the creation of a unified *Orchestrator* layer — a Catalyst-driven controller that will coordinate PowerShell scripts, Node.js modules and transformation functions from a single point. This will allow chained execution, smarter retry logic and a single dashboard view for end-to-end task status.


- Combine all Catalyst functions into unified “Orchestrator” function  
- Add PDF parsing templates for new agencies  
- Implement centralised logging dashboard in Zoho Analytics  
- Automate daily cleanup and archiving of Paradise folder  


## 10. Closing Notes & Acknowledgements  

**The Monster Manual – Data Transformation & PDF Processing System** began as a practical fix to a growing data challenge and evolved into a resilient, modular automation framework. What started with a few scripts and CSVs and later PDFs became a living ecosystem that watches, parses, transforms and learns.  

This document captures not just the system, but the mindset behind it — the pursuit of reliability through clarity, and creativity through code.  

### 💡 Acknowledgements  
A nod to the tools, patience and persistence that brought the Data Transformation Framework (DTF) to life:  
- **PowerShell**, for orchestrating the moving parts with precision.  
- **Node.js**, for bridging logic and performance.  
- **Zoho RPA and Catalyst**, for enabling deep automation without compromise.  

🤝 **Personal Thanks**  

- **Anthony Stevens** for helping me get back to work.

### 🎮 Dedication  
*In memory of my best friend, Archer Maclean — a true pioneer who showed that precision and fun belong side by side.*   
*His spirit of invention, humour and curiosity lives quietly between the lines of every function and every log entry.*  ![IK+ Character](https://github.com/GMJ2023/assets/blob/main/ikChar.png) 

---

**Appendix E** – Scheduled Tasks & Watchdogs
These Windows Scheduled Tasks act as the persistent background infrastructure for the Data Transformation Framework (DTF).
They ensure that critical services remain active and responsive without user intervention, and provide recovery instructions if a fault or stall occurs.

---

🕑 **Scheduled Task – Monitor Browser & Services**
Purpose:
Maintains operational awareness of both the Chrome automation browser and core service processes (file_manager_service.ps1 and Zoho RPA Agent).
The task ensures that essential DTF components continue running, sending alerts via Zoho Flow if anything stops or freezes.

Trigger & Configuration:

Type: Scheduled Task (Windows)
Trigger: Every 1 minute, indefinitely
Startup behaviour: Begins automatically at system boot
Runs whether user is logged in or not
Executes with highest privileges
Action & Target Script:

powershell.exe -ExecutionPolicy Bypass -File "C:\binary-proxy\monitor_browser.ps1"
Behaviour:

Checks that Chrome is running with the correct --remote-debugging-port=9222 flag
Confirms that file_manager_service.ps1 is producing a live heartbeat
Verifies Zoho RPA Agent process state
Issues a webhook alert if any service stops or freezes
Recovery Guidance:

Open Task Scheduler
Find FileMonitorService
If it’s already running, select End
Then select Run to restart
If it still fails, check the recent change log for system modifications
Notes:

Heartbeat window: 2 minutes
Grace period: 60 seconds at start-up
Self-healing logic prevents duplicate notifications
---

🕑 **Scheduled Task – Monitor Output Log**

Purpose:
Reviews recent DTF transformation logs and checks for stalled or empty outputs.
Designed to detect silent Catalyst failures where the system completes a run but produces a CSV with zero rows.

Trigger & Configuration:

Type: Scheduled Task (Windows)
Trigger: Every 15 minutes
Startup behaviour: Enabled at system boot
Runs silently in background context
Highest privileges enabled
Action & Target Script:

powershell.exe -ExecutionPolicy Bypass -File "C:\binary-proxy\monitor_output_log.ps1"
Behaviour:

Locates the latest transformation log file
Extracts the RowCount metric or similar indicators
Sends a webhook if record count equals zero or log inactivity exceeds 30 minutes
Recovery Guidance:

Check mapping templates and input files
Review Catalyst console logs
Re-run the affected transformation via run_transform.bat
Notes:

Maintains cooldown to prevent duplicate alerts
Continues operating during RPA restarts

## `Appendix` E – Scheduled Tasks & Watchdogs

These Windows Scheduled Tasks act as the persistent background infrastructure for the Data Transformation Framework (DTF).
They ensure that critical services remain active and responsive without user intervention, and provide recovery instructions if a fault or stall occurs.

---

### 🕑 Scheduled Task – Monitor Browser & Services

Purpose:
Maintains operational awareness of both the Chrome automation browser and core service processes (`file_manager_service.ps1` and Zoho RPA Agent).
The task ensures that essential DTF components continue running, sending alerts via Zoho Flow if anything stops or freezes.

Trigger & Configuration:

- Type: Scheduled Task (Windows)
- Trigger: Every 1 minute, indefinitely
- Startup behaviour: Begins automatically at system boot
- Runs whether user is logged in or not
- Executes with highest privileges

Action & Target Script:

```
powershell.exe -ExecutionPolicy Bypass -File "C:\binary-proxy\monitor_browser.ps1"
```

Behaviour:

- Checks that Chrome is running with the correct `--remote-debugging-port=9222` flag
- Confirms that `file_manager_service.ps1` is producing a live heartbeat
- Verifies Zoho RPA Agent process state
- Issues a webhook alert if any service stops or freezes

Recovery Guidance:

1. Open Task Scheduler
2. Find FileMonitorService
3. If it’s already running, select End
4. Then select Run to restart
5. If it still fails, check the recent change log for system modifications

Notes:

- Heartbeat window: 2 minutes
- Grace period: 60 seconds at start-up
- Self-healing logic prevents duplicate notifications

---

### 🕑 Scheduled Task – Monitor Output Log

Purpose:
Reviews recent DTF transformation logs and checks for stalled or empty outputs.
Designed to detect silent Catalyst failures where the system completes a run but produces a CSV with zero rows.

Trigger & Configuration:

- Type: Scheduled Task (Windows)
- Trigger: Every 15 minutes
- Startup behaviour: Enabled at system boot
- Runs silently in background context
- Highest privileges enabled

Action & Target Script:

```
powershell.exe -ExecutionPolicy Bypass -File "C:\binary-proxy\monitor_output_log.ps1"
```

Behaviour:

- Locates the latest transformation log file
- Extracts the RowCount metric or similar indicators
- Sends a webhook if record count equals zero or log inactivity exceeds 30 minutes

Recovery Guidance:

- Check mapping templates and input files
- Review Catalyst console logs
- Re-run the affected transformation via `run_transform.bat`

Notes:

- Maintains cooldown to prevent duplicate alerts
- Continues operating during RPA restarts

---

## 🧾 Change Log

- v10 – Added System Overview Diagram and updated dedication text (22 Oct 2025 – G.J.)
- v11 – Initial structured draft of the Monster Manual (21 Oct 2025 – G.J.)
- v12 – Editorial polish and structure refinement (22 Oct 2025 – G.J. & ChatGPT)
- v15 - End of Master Version — DTF15 (October 2025 – Geoffrey Jones)
- v15.1 – Added Appendix E (Scheduled Tasks & Watchdogs) including Monitor Browser & Services and Monitor Output Log definitions (Nov 2025 – G.J.)

---
*End of Master Version — DTF15 (November 2025 – Geoffrey Jones)*

---

**Copyright © 2025 Geoffrey Jones**
 
 **All rights reserved.**

This codebase and all associated materials are proprietary to Geoffrey Jones.
 No part of this work may be copied reproduced modified distributed or used for any purpose other than the specific engagement for which access was granted.
 Any other use requires prior written consent from the copyright holder.




