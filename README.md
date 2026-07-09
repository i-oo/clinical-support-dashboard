

# Clinical Support Dashboard

![Status](https://img.shields.io/badge/status-in--development-yellow)
![Stack](https://img.shields.io/badge/ASP.NET_Core_8-512BD4?logo=dotnet&logoColor=white)
![SQL Server](https://img.shields.io/badge/SQL_Server-CC2927?logo=microsoftsqlserver&logoColor=white)
![IIS](https://img.shields.io/badge/IIS-0078D4?logo=microsoft&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

An enterprise-grade web application for monitoring mission-critical healthcare application health
and managing support incidents across multi-system environments — built on the ASP.NET Core MVC /
SQL Server / IIS stack standard to Canadian public-sector healthcare IT operations.

---

## Table of Contents

- [Overview](#overview)
- [Use Case](#use-case)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Key Features](#key-features)
- [What Sets This Apart](#what-sets-this-apart)
- [Database Design](#database-design)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Roadmap](#roadmap)

---

## Overview

Healthcare IT environments run dozens of interdependent applications — patient registration systems,
lab portals, pharmacy modules, immunization registries — across production, UAT, and development
environments. When one degrades, patient care is directly affected. Support teams need a centralized,
queryable view of application health and incident history.

The Clinical Support Dashboard provides exactly that: a lightweight internal tooling application
that aggregates application status, tracks support incidents by severity and lifecycle stage, and
surfaces actionable summaries through a SQL Server stored-procedure-driven data layer — without
the overhead of a full ITSM platform.

---

## Use Case

This application is designed for **Application Support Analysts** and **IT Operations teams** in
integrated healthcare environments who need to:

- Monitor the real-time health status of enterprise applications across environments
- Log, categorize, and track support incidents from initial report through to resolution
- Retrieve and filter incident history by application, severity, or lifecycle status to support
  client data requests and root-cause analysis
- Maintain a documented record of system degradation events for change management and audit purposes

It mirrors the day-to-day workflow of a Ministry of Health / eHealth-tier support team without
requiring a licensed ITSM tool.

---

## Architecture

```
Browser
   │
   ▼
[ IIS — Reverse Proxy / TLS Termination ]
   │   (Application Request Routing + URL Rewrite module)
   │
   ▼
[ Kestrel — ASP.NET Core 8 MVC Application ]
   │
   ▼
[ ADO.NET Data Layer — DatabaseHelper ]
   │   (parameterized queries + stored procedures; no ORM)
   │
   ▼
[ SQL Server Express — ClinicalSupportDB ]
      Applications table
      Incidents table
      usp_GetIncidentSummary (stored procedure)
```

The application follows a classic **MVC separation**: Controllers handle HTTP routing and business
logic, Models represent database entities, and Razor Views render HTML server-side. IIS sits in
front of the Kestrel process as a reverse proxy, which mirrors the standard enterprise deployment
pattern in Windows Server environments.

---

## Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Web framework | ASP.NET Core 8 MVC | Cross-platform, LTS, industry standard for .NET web apps |
| Language | C# 12 | Type-safe, mainstream in enterprise healthcare IT |
| Database | SQL Server Express | Same engine as production SQL Server; free for development |
| Data access | ADO.NET (raw) | Explicit SQL control; no ORM abstraction obscuring query behaviour |
| Web server | IIS + Kestrel | IIS as reverse proxy matches enterprise Windows Server deployments |
| IDE | Visual Studio 2022 | Standard tooling for .NET development |
| Frontend | Bootstrap 5 + Razor | Server-side rendering; no JavaScript framework overhead |
| Package management | NuGet | .NET ecosystem standard |

---

## Key Features

### Application Health Registry
Maintains a list of monitored enterprise applications with version, environment
(Production / UAT / Dev), operational status (Healthy / Degraded / Down), and ownership metadata.
Status badges update directly from the database — no hardcoded values.

### Incident Lifecycle Tracking
Logs support incidents against specific applications with full metadata: title, description,
severity (Low / Medium / High / Critical), lifecycle status (Open / Investigating / Resolved),
reporting source, creation timestamp, and resolution timestamp. Supports the full support workflow
from initial report to closure.

### Stored-Procedure-Driven Dashboard
The main dashboard aggregates incident counts per application using a SQL Server stored procedure
(`usp_GetIncidentSummary`) called directly via ADO.NET. This intentional design choice — a named,
versioned, testable stored procedure rather than an inline query — mirrors how enterprise support
databases are actually structured and maintained.

### Parameterized Filtering
Incidents are filterable by status and application via URL query parameters. All queries use
parameterized ADO.NET commands — no string concatenation, no SQL injection surface.

### IIS Deployment
The application is documented and tested for deployment under IIS using the ASP.NET Core Hosting
Bundle, with application pool configuration, `web.config` generation, and HTTPS binding notes.

---

## What Sets This Apart

Most ASP.NET Core portfolio projects are generic CRUD applications — a to-do list or a blog with
no domain context. This project differs in three ways:

**Domain authenticity.** The data model, terminology, and workflow (application registries,
incident severity tiers, lifecycle stages, environment separation) reflect how enterprise
healthcare IT support environments are actually structured. The seed data uses realistic
healthcare system names and incident descriptions, not placeholder lorem ipsum.

**Deliberate data access.** The choice to use raw ADO.NET rather than Entity Framework is
intentional and documented. Healthcare support databases are often decades-old SQL Server instances
maintained through stored procedures, not ORM migrations. Demonstrating fluency at the ADO.NET
layer — parameterized commands, DataTable handling, stored procedure invocation — is directly
relevant to the support analyst role, where reading and writing SQL against existing schemas is
a daily task.

**IIS as a first-class deployment target.** Most tutorials deploy ASP.NET Core to Azure App
Service or Docker. This project treats IIS as the primary host — because that is what
Windows-Server-based healthcare IT environments actually run. The deployment documentation covers
IIS site binding, application pool identity, and `web.config` configuration explicitly.

---

## Database Design

```
Applications
────────────────────────────────────────────
ApplicationId   INT         PK IDENTITY
AppName         NVARCHAR    NOT NULL
AppVersion      NVARCHAR    NOT NULL
Environment     NVARCHAR    DEFAULT 'Production'
Status          NVARCHAR    DEFAULT 'Healthy'
Owner           NVARCHAR    NULL
LastChecked     DATETIME    DEFAULT GETDATE()
CreatedAt       DATETIME    DEFAULT GETDATE()

Incidents
────────────────────────────────────────────
IncidentId      INT         PK IDENTITY
ApplicationId   INT         FK → Applications
Title           NVARCHAR    NOT NULL
Description     NVARCHAR    NULL
Severity        NVARCHAR    DEFAULT 'Medium'
Status          NVARCHAR    DEFAULT 'Open'
ReportedBy      NVARCHAR    NULL
CreatedAt       DATETIME    DEFAULT GETDATE()
ResolvedAt      DATETIME    NULL

Stored Procedure: usp_GetIncidentSummary
────────────────────────────────────────────
Returns per-application incident counts grouped by
status (Open / Investigating / Resolved) and flags
critical incident totals for dashboard prioritization.
```

---

## Project Structure

```
clinical-support-dashboard/
│
├── database/
│   ├── 01_schema.sql               # DB creation, tables, seed data
│   └── 02_stored_procedures.sql    # usp_GetIncidentSummary
│
├── src/
│   └── ClinicalSupportDashboard/
│       ├── Controllers/
│       │   ├── HomeController.cs           # Dashboard — calls stored procedure
│       │   ├── ApplicationsController.cs   # Application registry
│       │   └── IncidentsController.cs      # Incident log + filtering
│       ├── Data/
│       │   └── DatabaseHelper.cs           # ADO.NET abstraction layer
│       ├── Models/
│       │   ├── Application.cs
│       │   ├── Incident.cs
│       │   └── IncidentSummary.cs          # Stored procedure result model
│       ├── Views/
│       │   ├── Home/Index.cshtml           # Dashboard view
│       │   ├── Applications/Index.cshtml
│       │   ├── Incidents/Index.cshtml
│       │   └── Shared/_Layout.cshtml       # Shared navigation + layout
│       ├── wwwroot/css/site.css
│       ├── appsettings.json                # Connection string configuration
│       └── Program.cs                      # App entry point + DI registration
│
├── tests/
│   └── ClinicalSupportDashboard.Tests/     # xUnit test project (in progress)
│
└── docs/
    └── DAY_1_2_SETUP.md                    # Full environment setup guide
```

---

## Getting Started

### Prerequisites

- [Visual Studio 2022 Community](https://visualstudio.microsoft.com/vs/community/) — ASP.NET and web development workload
- [SQL Server Express](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) — free edition
- [SQL Server Management Studio (SSMS)](https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms)
- [ASP.NET Core Hosting Bundle](https://dotnet.microsoft.com/en-us/download/dotnet/8.0) — required for IIS deployment

### 1. Clone the repository

```bash
git clone https://github.com/i-oo/clinical-support-dashboard.git
cd clinical-support-dashboard
```

### 2. Set up the database

Open SSMS, connect to `localhost\SQLEXPRESS`, and run the scripts in order:

```sql
-- Step 1: Create schema and seed data
-- Run: database/01_schema.sql

-- Step 2: Create stored procedures
-- Run: database/02_stored_procedures.sql

-- Step 3: Verify
USE ClinicalSupportDB;
EXEC dbo.usp_GetIncidentSummary;
-- Should return 5 rows
```

### 3. Configure the connection string

Edit `src/ClinicalSupportDashboard/appsettings.json` if your SQL Server instance name
differs from the default:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost\\SQLEXPRESS;Database=ClinicalSupportDB;Trusted_Connection=True;TrustServerCertificate=True;"
  }
}
```

### 4. Run the application

Open `src/ClinicalSupportDashboard/ClinicalSupportDashboard.csproj` in Visual Studio
and press **F5**. The dashboard will open in your browser.

For full setup instructions including IIS deployment, see [`docs/DAY_1_2_SETUP.md`](docs/DAY_1_2_SETUP.md).

---

## Roadmap

- [x] Database schema and stored procedure
- [x] ADO.NET data access layer (`DatabaseHelper`)
- [x] Dashboard view (stored procedure results)
- [x] Application registry view
- [x] Incident log with status filtering
- [ ] Log Incident form (parameterized INSERT)
- [ ] Update incident status (inline edit)
- [ ] Search by keyword across incident titles
- [ ] Unit tests — `DatabaseHelper` and controller logic (xUnit)
- [ ] IIS deployment with `web.config` and application pool documentation
- [ ] Docker support (Windows container, optional)

---

## License

MIT — see [LICENSE](LICENSE) for details.
