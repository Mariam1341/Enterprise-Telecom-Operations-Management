# Enterprise Telecom Operations Management Platform

A ServiceNow scoped application that automates the end-to-end telecom customer operations lifecycle — from customer onboarding and complaint intake to incident escalation, network outage tracking, field engineer dispatch, SLA-driven resolution, and manager-facing dashboards.

Built as an ITI (Information Technology Institute) graduation project on the ServiceNow platform, covering CSA fundamentals through applied development: data modeling, security (ACLs), client/server scripting, Flow Designer automation, SLA management, Service Portal delivery, and reporting.

![ServiceNow](https://img.shields.io/badge/Platform-ServiceNow-00cc66?style=flat-square)
![Scope](https://img.shields.io/badge/App%20Scope-x__telcom__ops-032B43?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-005955?style=flat-square)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Key Design Decision — Employee Service](#key-design-decision--employee-service)
- [Application Modules](#application-modules)
- [Data Model](#data-model)
- [Roles & Groups](#roles--groups)
- [Automation (Flow Designer)](#automation-flow-designer)
- [Security (ACL)](#security-acl)
- [SLA, Events & Notifications](#sla-events--notifications)
- [Service Portal](#service-portal)
- [Reports & Dashboard](#reports--dashboard)
- [Build Order (Implementation Phases)](#build-order-implementation-phases)
- [End-to-End Demo Flow](#end-to-end-demo-flow)
- [Governance Rules](#governance-rules)
- [Documentation](#documentation)
- [Tech Stack](#tech-stack)
- [Project Status & Future Work](#project-status--future-work)
- [Author](#author)

---

## Overview

| Property | Value |
|---|---|
| **Project Name** | Enterprise Telecom Operations Management Platform |
| **Platform** | ServiceNow Personal Developer Instance (PDI) |
| **Application Scope** | `x_telcom_ops` |
| **Application Type** | Scoped Application |
| **Track** | ServiceNow Fundamentals + Development (CSA + CAD + ITSM + Portal) |

Customers can raise telecom complaints, complaint agents manage and resolve them, incidents are created automatically, network outages are tracked and correlated with complaints, field engineers receive and resolve repair tasks, SLAs monitor resolution time, notifications trigger automatically at every milestone, and reports/dashboards give operations managers real-time KPI visibility.

## Problem Statement

Telecom operators typically struggle with:

- High volumes of customer complaints (network outages, slow internet, dropped calls, billing issues, SIM problems) tracked across disconnected spreadsheets, calls, or emails.
- Network outages that aren't centrally tracked or correlated with the complaints they cause.
- Slow resolution due to manual assignment of complaints and repair tasks.
- No formal SLA mechanism to measure whether critical issues are resolved on time.
- No unified, real-time view for operations managers.

This project addresses each of these using native ServiceNow capabilities — no custom, hard-to-maintain scripting required.

## Key Design Decision — Employee Service

> **Note:** Unlike a fully open, unauthenticated "Customer Registration" portal form, this implementation registers customers through an **Employee Service** instead. An authorised employee (e.g. a complaint agent) creates the Customer Profile record on the customer's behalf via a Service Catalog item, keeping onboarding under employee control for data quality, identity verification, and duplicate prevention. Every other customer-facing flow — raising a complaint, tracking complaint/outage status, and requesting compensation — remains fully self-service on the Telecom Customer Portal, exactly as originally specified.

## Application Modules

| Module | Description |
|---|---|
| Customer Management | Store telecom customer profiles |
| Service Plan Management | Telecom plans catalog |
| Complaint Management | Manage the complaint lifecycle |
| Incident Management | ITSM integration via auto-created Incidents |
| Network Outage Management | Outage tracking and correlation |
| Field Engineer Management | Dispatch and engineer task tracking |
| Compensation Management | Customer compensation requests & approval |
| Reporting & Dashboard | KPI monitoring |

## Data Model

Six custom tables under the `x_telcom_ops` scope. Three extend the out-of-the-box **Task** table (inheriting `Number`, `State`, `Priority`, `Assigned To`, `Assignment Group`, `Comments`, `Work Notes` — never duplicated).

| # | Table | Extends | Purpose |
|---|---|---|---|
| 1 | `x_telcom_customer_profile` | — | Customer master data |
| 2 | `x_telcom_service_plan` | — | Telecom service plans catalog |
| 3 | `x_telcom_customer_complaint` | Task | Customer complaints → auto-escalates to Incident |
| 4 | `x_telcom_network_outage` | Task | Network outage tracking |
| 5 | `x_telcom_field_engineer_task` | Task | Field engineer repair work |
| 6 | `x_telcom_compensation` | — | Customer compensation requests |

**Relationship chain:** `Customer → Complaint → Incident → Network Outage → Field Engineer Task`, with `Compensation` linked back to both `Customer` and `Complaint`.

> Full field-level dictionaries for every table are in the [documentation](#documentation).

## Roles & Groups

| Role | Group | Responsibility |
|---|---|---|
| `x_telcom_ops.customer` | Telecom Customers | Portal access |
| `x_telcom_ops.complaint_agent` | Complaint Resolution Team | Handle complaints |
| `x_telcom_ops.network_engineer` | Network Operations Team | Manage outages |
| `x_telcom_ops.field_engineer` | Field Engineering Team | Resolve engineer tasks |
| `x_telcom_ops.operations_manager` | Telecom Operations Team | Approvals & dashboard |
| `itil` | ITSM Support Team | Incident handling |
| `admin` | — | Full access |

## Automation (Flow Designer)

| Flow | Trigger | Actions |
|---|---|---|
| Flow 1 — Customer Registration | Customer created | Create `sys_user`, assign role, send welcome notification |
| Flow 2 — Complaint Flow | Complaint created | Set priority, assign group, create incident, notify |
| Flow 3 — Incident Flow | Incident created | Assign ITSM Support Team, notify |
| Flow 4 — Outage Flow | Outage created | Assign Network Ops Team, create engineer task, notify |
| Flow 5 — Engineer Flow | Engineer task updated | Update outage state, close outage |
| Flow 6 — Compensation Flow | Compensation created | Send approval, notify customer |

Complemented by six Business Rules (auto-assignment, duplicate prevention, auto incident/engineer-task creation, auto outage closure), three Client Scripts (mobile validation, auto priority, engineer validation), two UI Policies, two Data Policies, and three Script Includes (`CustomerUtils`, `ComplaintUtils`, `EngineerUtils`) exposed via GlideAjax.

## Security (ACL)

| Table | Access Rule |
|---|---|
| Customer | Customers can see only their own record |
| Complaint | Complaint Resolution Team only |
| Outage | Network Operations Team only |
| Engineer Task | Field Engineering Team only |
| Compensation | Operations Manager only |

## SLA, Events & Notifications

**Complaint SLA** — Start: `State = Open` · Pause: `State = Pending` · Stop: `State = Resolved/Closed`

| Priority | SLA Time |
|---|---|
| Critical | 1 Hour |
| High | 4 Hours |
| Medium | 8 Hours |
| Low | 24 Hours |

**Outage SLA** — Start: `State = Open` · Stop: `State = Resolved`

| Severity | SLA Time |
|---|---|
| Critical | 30 Minutes |
| High | 2 Hours |
| Medium | 4 Hours |
| Low | 8 Hours |

**Events:** `customer.created`, `complaint.created`, `incident.created`, `outage.created`, `engineer.assigned`, `compensation.approved` — each triggers a corresponding email notification.

## Service Portal

**Telecom Customer Portal** — built using only Record Producers, the Data Table widget, and the Simple List widget (no advanced Angular widgets, per project rules).

| Page | Component |
|---|---|
| Home | Welcome, Quick Links, My Complaints, Active Outages widgets |
| Employee Service — Customer Registration | Record Producer (employee-facing) |
| Raise Complaint | Record Producer |
| My Complaints | Data Table Widget |
| Outage Status | Simple List Widget |
| Compensation Request | Record Producer |

## Reports & Dashboard

**Telecom Operations Dashboard** — Open Complaints (KPI), Active Outages (KPI), SLA Breaches (Bar Chart), Engineer Workload (Pie Chart), Complaint Trend (Line Chart).

**Standalone reports:** Complaints by Type, Open Outages, Engineer Task Status, SLA Breach Report, Complaint Trend Report.

## Build Order (Implementation Phases)

Development followed a strict, dependency-aware order:

1. Application Setup (scope, roles, groups, menu)
2. Data Model Setup
3. Forms & UI Configuration
4. Security (ACL)
5. Client-Side Scripting
6. Server-Side Scripting (UI/Data Policies, Business Rules, Script Includes)
7. Automation (Flow Designer)
8. SLA + Events + Notifications
9. Service Portal
10. Reports & Dashboard
11. Final Demo

## End-to-End Demo Flow

```
Customer Registration → Portal Login → Raise Complaint → Complaint Assignment
→ Incident Creation → Outage Creation → Engineer Task Creation
→ Engineer Resolution → Outage Closure → SLA Tracking
→ Notification Trigger → Dashboard Update
```

## Governance Rules

- Reuse existing Task fields — never duplicate them.
- Keep Business Rules simple.
- Keep Flow Designer flows to a maximum of 3–5 actions.
- Use Record Producers in the Service Portal; avoid advanced/complex scripting.
- Test every module separately, then test the full chain (SLA, Notifications, Assignments, Portal submission, Incident creation) before the final demo.

## Documentation

Full project documentation — including complete data dictionaries, ACL definitions, script bodies, test cases, results & discussion, and screenshots of every module — is available here:

📄 [`/docs/Telecom_Project_Documentation.docx`](./docs/Telecom_Project_Documentation.docx)

## Tech Stack

- **Platform:** ServiceNow (Personal Developer Instance)
- **Development:** Scoped Application, Flow Designer, Business Rules, Client Scripts, Script Includes, GlideAjax
- **Frontend:** ServiceNow Service Portal (Record Producers, Data Table Widget, Simple List Widget)
- **Reporting:** ServiceNow Reports & Performance Analytics-style Dashboards

## Project Status & Future Work

✅ Completed — all 11 phases implemented and demoed end-to-end.

Planned enhancements:

- AI-based complaint classification and outage root-cause suggestions
- Real network monitoring (NMS) integration via REST
- Mobile app support for field engineers
- Optional verified customer self-registration alongside the Employee Service
- Performance Analytics for long-term SLA/outage trend analysis
- Billing system integration for automated compensation payouts

---

*This project was built for educational purposes as part of the ITI graduation program and runs on a ServiceNow PDI (Personal Developer Instance).*
