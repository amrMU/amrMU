# EftahShop — Production Architecture & Engineering Case Study

A production-oriented **multi-tenant e-commerce SaaS platform** built with Laravel for merchants operating across Egypt and Saudi Arabia.

This document focuses on the backend architecture, infrastructure decisions, reliability patterns, and production engineering challenges behind the platform.

> The EftahShop source code is proprietary. This public case study intentionally focuses on architecture and engineering decisions without exposing private implementation details, credentials, or business-sensitive code.

---

# Overview

EftahShop provides merchants with isolated online stores and a centralized platform for managing:

* Products and inventory
* Orders and customers
* Storefront configuration
* Discounts and coupons
* Shipping workflows
* Merchant payments
* Platform commissions
* Merchant balances
* Custom domains
* Store provisioning

The engineering goal is to keep the system simple enough to operate as a Laravel application while maintaining clear boundaries around tenant isolation, financial workflows, asynchronous processing, and infrastructure concerns.

---

# High-Level Architecture

```text
                         ┌──────────────────────────┐
                         │     Central Platform     │
                         │                          │
                         │  Merchants               │
                         │  Domains                 │
                         │  Platform Finance        │
                         │  Payment Configuration   │
                         │  Tenant Provisioning     │
                         └────────────┬─────────────┘
                                      │
                         Tenant Resolution
                                      │
               ┌──────────────────────┼──────────────────────┐
               │                      │                      │
               ▼                      ▼                      ▼

        ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
        │  Merchant A │        │  Merchant B │        │  Merchant N │
        │  Database   │        │  Database   │        │  Database   │
        └──────┬──────┘        └──────┬──────┘        └──────┬──────┘
               │                      │                      │
        Storefront             Storefront             Storefront
        Products               Products               Products
        Inventory              Inventory              Inventory
        Orders                 Orders                 Orders
        Customers              Customers              Customers
        Payments               Payments               Payments
```

---

# Multi-Tenant Architecture

EftahShop uses a **database-per-tenant** model.

Each merchant operates inside an isolated tenant environment while platform-level data remains in a central database.

## Central responsibilities

The central application manages platform-level concerns such as:

* Merchant accounts
* Tenant provisioning
* Platform domains and custom domains
* Platform financial configuration
* Merchant platform balances
* Commission rules
* Payment provider configuration

## Tenant responsibilities

Each tenant owns its commerce data, including:

* Products
* Categories
* Inventory
* Customers
* Orders
* Coupons
* Store configuration
* Customer payment records

## Why database-per-tenant?

The architecture prioritizes:

* Strong merchant data isolation
* Clear tenant ownership
* Reduced accidental cross-tenant access
* Easier tenant-specific operations
* A clear path for future database distribution

The main tradeoff is increased operational complexity around migrations, provisioning, tenant-aware jobs, and centralized reporting.

---

# Application Architecture

The backend follows a layered structure where useful:

```text
HTTP Request
     │
     ▼
Controller
     │
     ▼
Request Validation
     │
     ▼
DTO / Application Input
     │
     ▼
Service / Domain Workflow
     │
     ├──────────► Repository / Query Layer
     │
     ├──────────► External Provider
     │
     └──────────► Event / Queue
     │
     ▼
API Resource / Response
```

The objective is not to add abstraction everywhere.

Simple operations remain simple, while workflows involving payments, finance, tenant boundaries, provisioning, or multiple state transitions are moved into dedicated application services.

---

# Commerce & Order Workflows

The tenant commerce layer manages the full order lifecycle.

```text
Customer
   │
   ▼
Storefront
   │
   ▼
Checkout
   │
   ├── Validate product availability
   ├── Calculate totals
   ├── Apply discounts
   ├── Validate shipping
   └── Select payment method
   │
   ▼
Order
   │
   ├── Pending
   ├── Confirmed
   ├── Preparing
   ├── Shipped
   └── Delivered
```

Cancelled orders follow a separate terminal path.

The order layer is also responsible for coordinating inventory changes, payment state, merchant workflows, and platform commission rules.

---

# Inventory Consistency

Inventory is treated as part of the transactional order workflow rather than only as a display value.

The system needs to prevent situations such as:

```text
Available quantity: 1

Customer A ───────► Checkout
Customer B ───────► Checkout

Both should not successfully purchase
the final unit.
```

Critical stock operations are designed around database consistency and transactional boundaries.

The goal is to ensure that product availability, order creation, and inventory changes cannot silently diverge during concurrent requests or failed operations.

---

# Payment Architecture

Payment handling is separated from core order logic through gateway-oriented abstractions.

```text
Application
     │
     ▼
Payment Service
     │
     ▼
Gateway Contract
     │
     ├────► Paymob
     ├────► Manual Bank Transfer
     ├────► InstaPay
     └────► Mobile Wallet
```

This prevents controllers and business workflows from becoming tightly coupled to one payment provider.

Payment workflows support different operational models:

### Online payments

```text
Order
  │
  ▼
Initialize Payment
  │
  ▼
External Gateway
  │
  ▼
Callback / Verification
  │
  ▼
Payment Confirmed
```

### Manual payments

Merchants can configure manual payment methods while payment evidence and confirmation are handled separately from gateway-based payments.

### Cash on delivery

Commission handling can depend on the successful delivery of the order rather than order creation.

---

# Platform Commission & Merchant Balance

EftahShop uses a commission-based platform model rather than requiring a monthly merchant subscription.

Platform finance is separated from the tenant's customer-facing order data.

```text
Customer Order
      │
      ▼
Commission Rule
      │
      ├──── Online Payment ───► Commission on successful payment
      │
      └──── COD ──────────────► Commission on delivery
      │
      ▼
Merchant Platform Balance
```

This required treating platform accounting as its own domain rather than adding financial calculations directly inside order controllers.

---

# Queue & Async Processing

Not every operation should block an HTTP request.

Queues are used for workflows where asynchronous execution improves reliability or request latency.

Examples include:

* Notifications
* Payment-related processing
* Tenant provisioning tasks
* Background synchronization
* Operational side effects

A typical workflow looks like:

```text
Request
   │
   ▼
Database Transaction
   │
   ▼
Business State Committed
   │
   ▼
Event
   │
   ▼
Queued Job
   │
   ├── Retry
   ├── Failure Handling
   └── External Integration
```

The important boundary is that critical business state should not depend on an unreliable external network call completing inside the original request whenever that dependency can safely be asynchronous.

---

# Redis

Redis is used where in-memory infrastructure provides a clear operational benefit rather than as a default storage layer.

Typical responsibilities include:

* Caching
* Queue infrastructure
* Temporary application state
* Rate-sensitive operations
* Reducing unnecessary database work

Persistent business data remains in the appropriate relational database.

---

# Custom Domains & TLS Automation

Each merchant can operate using an EftahShop subdomain and the architecture also supports merchant-owned custom domains.

This creates an infrastructure workflow beyond normal Laravel routing:

```text
DNS
 │
 ▼
Cloudflare
 │
 ▼
Caddy
 │
 ▼
TLS Negotiation
 │
 ▼
Domain Authorization
 │
 ▼
Tenant Resolution
 │
 ▼
Laravel
```

Domain state is tracked centrally so infrastructure decisions can be made using trusted platform data.

---

# Production Incident: On-Demand TLS

One of the more interesting production incidents occurred in the automated TLS provisioning flow.

## Symptom

Newly provisioned merchant domains began timing out before requests reached the Laravel application.

Laravel itself was healthy.

That meant the issue had to be traced through the infrastructure path:

```text
DNS
→ Cloudflare
→ Caddy
→ TLS
→ Laravel
```

## Root Cause

Caddy was using On-Demand TLS for dynamic hostnames.

The authorization endpoint configured for certificate issuance pointed to a generic application health endpoint.

That health endpoint always returned HTTP `200`.

Effectively, Caddy was being told:

```text
Any requested hostname is authorized.
```

This allowed certificate requests for hostnames that had never been registered by the platform and eventually contributed to certificate issuance limits being exhausted.

## Fix

The TLS authorization flow was redesigned.

A dedicated internal authorization endpoint now validates the requested hostname against the central platform domain records before Caddy is allowed to request a certificate.

```text
Caddy
  │
  │  Can I provision TLS for example.com?
  ▼
Internal Authorization Endpoint
  │
  ▼
Central Domain Database
  │
  ├── Registered + Valid ─────► HTTP 200
  │
  └── Unknown / Invalid ──────► HTTP 404
```

Platform subdomains were also moved toward wildcard certificate handling so certificates do not need to be individually issued for every merchant subdomain.

## Result

The new design:

* Prevents arbitrary hostname certificate issuance
* Protects certificate provisioning limits
* Separates domain authorization from application health checks
* Makes domain onboarding more predictable
* Keeps custom-domain routing compatible with the tenant system

## Engineering Lesson

The important part of this incident was not the final code change.

It required debugging the full request lifecycle across:

**DNS → CDN → reverse proxy → TLS → tenant resolution → application**

instead of assuming that a timeout originated inside Laravel.

---

# Tenant Provisioning

Creating a store involves more than inserting a merchant row.

A provisioning workflow needs to coordinate:

```text
Merchant Registration
        │
        ▼
Tenant Record
        │
        ▼
Tenant Database
        │
        ▼
Migrations / Initial Data
        │
        ▼
Domain Registration
        │
        ▼
Store Configuration
        │
        ▼
Ready Store
```

Provisioning-related operations are kept separate from normal request handling so the workflow can evolve independently as infrastructure requirements grow.

---

# Reliability Principles

Several principles guide the platform architecture.

### Keep financial state explicit

Payment status, order status, commission state, and platform balance are different concepts and should not be represented by one generic state.

### Commit business state before side effects

Where possible:

```text
Persist state
→ commit transaction
→ dispatch async work
```

rather than letting external providers control the success of the main database workflow.

### Treat infrastructure as part of the system

Production failures may exist outside PHP.

Debugging therefore includes:

* DNS
* TLS
* proxies
* container networking
* queues
* databases
* external providers

### Prefer incremental architecture

The platform remains a Laravel-based application rather than being prematurely decomposed into microservices.

Complexity is introduced when the problem requires it.

---

# Scaling Strategy

The architecture is intended to scale incrementally.

## Current foundation

```text
Laravel Application
       │
       ├── MySQL
       ├── Redis
       ├── Queue Workers
       ├── Caddy
       └── Docker
```

## Growth path

Individual infrastructure components can be scaled independently when load requires it:

```text
                     Load Balancer
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
         App #1         App #2         App #N

                 Shared Redis
                      │
              Queue Workers
                      │
           Tenant Databases
```

Possible scaling steps include:

* Horizontal application scaling
* Dedicated queue workers
* Independent Redis infrastructure
* Database workload separation
* Centralized observability
* Dedicated workloads for expensive background processes

The guiding principle is to scale based on measured bottlenecks rather than introducing distributed-system complexity prematurely.

---

# Technology Stack

### Backend

* PHP
* Laravel
* REST APIs
* stancl/tenancy

### Data

* MySQL
* Redis

### Infrastructure

* Docker
* Caddy
* Cloudflare
* Linux

### Architecture

* Multi-Tenancy
* Queue-Driven Workflows
* Service-Oriented Application Design
* Payment Gateway Abstraction
* Event-Driven Processing

---

# Key Engineering Areas Demonstrated

This project represents practical experience with:

* Designing multi-tenant SaaS systems
* Maintaining tenant data isolation
* Building production commerce workflows
* Designing payment abstractions
* Financial state management
* Queue-based processing
* Redis-backed infrastructure
* Domain and TLS automation
* Debugging distributed request paths
* Production incident investigation
* Incremental scaling
* Laravel application architecture

---

# Project Status

**EftahShop is a live, actively developed product.**

Platform:

https://eftahshop.com

The implementation remains proprietary, while this document serves as a public engineering case study of selected architecture and production challenges.
