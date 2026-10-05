# Smart Parking Allocation System

> **Algorithm-driven parking management system built with Java 21, Spring Boot, Spring Security, JPA, H2, and a vanilla HTML/CSS/JavaScript frontend.**

[![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3.x-brightgreen?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![Spring Security](https://img.shields.io/badge/Spring%20Security-RBAC-green?logo=springsecurity&logoColor=white)](https://spring.io/projects/spring-security)
[![Maven](https://img.shields.io/badge/Build-Maven-C71A36?logo=apachemaven&logoColor=white)](https://maven.apache.org/)
[![Database](https://img.shields.io/badge/Database-H2-blue)](https://www.h2database.com/)
[![Testing](https://img.shields.io/badge/Testing-JUnit%205-25A162?logo=junit5&logoColor=white)](https://junit.org/junit5/)

---

## 1. Project Overview

The **Smart Parking Allocation System** is a backend-driven parking management application designed around one central principle:

> **A vehicle never chooses its own parking slot. The system determines the safest and most appropriate slot algorithmically.**

Instead of treating parking as a simple CRUD operation, the project models the problem as an **allocation, scheduling, queuing, search, and access-control problem**.

The system:

- automatically selects compatible parking slots;
- prioritizes waiting vehicles using a priority queue;
- prevents incompatible vehicle-slot assignments;
- automatically reallocates released parking slots;
- maintains recent exit history using stack semantics;
- provides fast vehicle lookup using hashing;
- exposes separate interfaces for administrators, operators, and viewers;
- applies role-based authorization at the server level;
- prevents sensitive vehicle-owner information from reaching viewer APIs;
- validates application behavior with automated tests.

The repository is intentionally structured so that the **data structures and allocation algorithms are independent from the web and persistence layers**.

---

## 2. Why This Project Is Different

Many student parking projects primarily demonstrate:

```text
Vehicle → Select Slot → Save Record
```

This project instead models:

```text
Vehicle Arrival
      │
      ▼
Validate Request
      │
      ▼
Check Existing Parking / Waiting State
      │
      ▼
Compatibility Filtering
      │
      ▼
Slot Ranking
      │
      ├── Compatible Slot Found
      │        │
      │        ▼
      │     Allocate
      │
      └── No Compatible Slot
               │
               ▼
        Priority Queue
               │
               ▼
       Wait Until Capacity
               │
               ▼
        Automatic Reallocation
```

The engineering objective is therefore not only to **store parking records**, but to make the allocation decision **deterministic, explainable, and testable**.

---

# 3. Core Features

## Intelligent Parking Allocation

Parking slots are not manually selected by the vehicle.

The allocation engine evaluates:

1. slot availability;
2. vehicle-slot compatibility;
3. slot preference;
4. slot number ordering;
5. final allocation decision.

Example:

```text
SUV
 │
 ├── NORMAL slot → incompatible
 ├── EV slot → incompatible
 ├── ACCESSIBLE slot → incompatible
 └── LARGE slot → compatible
                         │
                         ▼
                    Allocate
```

This prevents physically inappropriate assignments such as placing an SUV in a compact bay.

---

## Priority-Based Waiting Queue

When no compatible parking slot is available, the vehicle enters a `PriorityQueue`.

Ordering follows:

```text
EMERGENCY
    ↓
HIGH
    ↓
NORMAL
```

For vehicles with equal priority, earlier arrival takes precedence.

The implementation therefore combines:

- priority ordering;
- FIFO semantics for equal priority;
- deterministic sequence-based tie breaking.

Core implementation:

```text
WaitingVehiclePriorityQueue
```

---

## Automatic Reallocation

A released parking slot is immediately evaluated against the waiting queue.

Example:

```text
A-04 becomes AVAILABLE
        │
        ▼
Inspect waiting queue
        │
        ▼
Find highest-priority compatible vehicle
        │
        ▼
Allocate A-04
        │
        ▼
Update parking session
        │
        ▼
Record audit event
```

A key design decision is:

> **Compatibility is evaluated before priority-based allocation.**

A high-priority vehicle cannot receive a physically incompatible bay merely because its priority is higher.

---

## Vehicle Lookup

Vehicles are indexed through a hash-based lookup structure.

```text
Vehicle Number
      │
      ▼
Normalize
      │
      ▼
HashMap
      │
      ▼
Average O(1) lookup
```

This is useful for:

- duplicate detection;
- vehicle search;
- gate operations;
- active-session checks.

Implementation:

```text
VehicleLookupManager
```

---

## Recent Exit History

Recent departures are modelled using LIFO semantics with `ArrayDeque`.

```text
Latest Exit
     ↓
   TOP
     ↓
Previous Exit
     ↓
Older Exit
```

The stack is bounded to prevent unlimited in-memory growth.

Implementation:

```text
ParkingHistoryStack
```

Default capacity:

```text
50 records
```

---

## Role-Based Access Control

The application separates users into three operational roles.

| Role | Responsibility |
|---|---|
| `SUPER_ADMIN` | Complete administrative control |
| `OPERATOR` | Gate and parking operations |
| `VIEWER` | Read-only operational visibility |

Architecture:

```text
                    ┌───────────────────┐
                    │    SUPER_ADMIN    │
                    │ Full Control      │
                    └─────────┬─────────┘
                              │
               ┌──────────────┴──────────────┐
               ▼                             ▼
       ┌────────────────┐            ┌────────────────┐
       │    OPERATOR    │            │     VIEWER     │
       │ Gate Operations│            │ Read Only      │
       └────────────────┘            └────────────────┘
```

Server-side access rules are enforced through Spring Security.

---

# 4. Security Architecture

Security is implemented as a backend responsibility rather than relying only on frontend route hiding.

## Authentication

The application uses:

- Spring Security;
- BCrypt password hashing;
- session-based authentication;
- `HttpOnly` session cookie configuration;
- CSRF protection.

---

## API Authorization

Protected API namespaces include:

```text
/api/admin/**
/api/operator/**
/api/viewer/**
```

Example authorization model:

```text
SUPER_ADMIN
    ├── /api/admin/**
    ├── /api/operator/**
    └── /api/viewer/**

OPERATOR
    ├── /api/operator/**
    └── /api/viewer/**

VIEWER
    └── /api/viewer/**
```

Unauthorized access is rejected server-side.

---

## Data Exposure Control

The viewer interface is deliberately separated from administrative and operator DTOs.

Viewer responses do not expose fields such as:

```text
ownerName
contact
raw vehicleNumber
```

Instead, public-facing vehicle identifiers can be represented in masked form.

This follows a **least-privilege data exposure model**:

```text
Database Entity
      │
      ▼
Role-specific DTO
      │
      ├── Admin DTO
      ├── Operator DTO
      └── Viewer DTO
```

The entity itself is not returned directly to clients.

---

## CSRF Protection

Spring Security CSRF protection is enabled using a cookie-based token repository and a custom request handler suitable for JavaScript API requests.

The implementation also returns structured JSON error responses instead of exposing Java/Spring exception pages.

---

## Brute-Force Protection

The application includes login attempt lockout behavior.

The implemented security tests verify that repeated failed authentication attempts trigger account lockout behavior.

---

# 5. Data Structures & Algorithms

One of the primary objectives of this project is demonstrating practical application of Data Structures and Algorithms.

| Concept | Implementation | Complexity | Purpose |
|---|---|---:|---|
| HashMap | `VehicleLookupManager` | Average O(1) lookup | Vehicle lookup / duplicate detection |
| PriorityQueue / Binary Heap | `WaitingVehiclePriorityQueue` | O(log n) insertion/removal | Priority-based waiting |
| ArrayDeque / Stack | `ParkingHistoryStack` | O(1) push/pop | Recent exit history |
| Linear Search | `SlotSearchManager` | O(n) | Slot search demonstration |
| Comparator | `SlotRankComparator` | O(k log k) sorting | Candidate slot ranking |
| EnumMap | `ParkingAllocationManager` | O(1) average preference access | Vehicle-slot preference matrix |

---

# 6. Vehicle Compatibility Model

The allocation engine uses an explicit compatibility matrix.

| Vehicle Type | Preferred Slot | Fallback | Restriction |
|---|---|---|---|
| `CAR` | `NORMAL` | `LARGE` | Incompatible types rejected |
| `EV` | `EV` | `NORMAL` | EV bay preferred |
| `SUV` | `LARGE` | None | Large bay required |
| `ACCESSIBLE` | `ACCESSIBLE` | `NORMAL` | Accessible bay preferred |

The important rule is:

```text
COMPATIBILITY
      ↓
PRIORITY
      ↓
SLOT RANKING
```

Priority never overrides physical compatibility.

---

# 7. Allocation Algorithm

The core allocation flow is implemented in:

```text
ParkingAllocationManager
```

Conceptually:

```java
for each parking slot:
    if slot is AVAILABLE:
        if slot is compatible with vehicle:
            add to candidate list

if candidate list is empty:
    return WAITING

sort compatible candidates

select best-ranked candidate

return allocation decision
```

### Complexity

For `n` total slots and `k` compatible candidates:

```text
Filtering       → O(n)
Sorting         → O(k log k)
Overall         → O(n + k log k)
```

Since `k <= n`:

```text
Worst case ≈ O(n log n)
```

---

# 8. Automatic Reallocation Algorithm

When a vehicle exits:

```text
1. Close active parking session
2. Record exit timestamp
3. Calculate parking duration
4. Release parking slot
5. Inspect waiting PriorityQueue
6. Search for compatible waiting vehicle
7. Temporarily defer incompatible entries
8. Assign released slot to compatible vehicle
9. Restore deferred entries
10. Update session state
11. Record audit event
```

This avoids a subtle queue bug where an incompatible vehicle at the head would otherwise block every compatible vehicle behind it.

---

# 9. Persistence Model

The project uses:

```text
Spring Data JPA
        │
        ▼
       H2
```

Persistent domain concepts include:

- users;
- vehicles;
- parking slots;
- parking sessions;
- audit records.

The in-memory waiting queue is rebuilt from persisted waiting-session data during startup.

This gives the application a recovery path for waiting vehicles after an application restart.

---

# 10. Project Architecture

The backend follows a layered Spring Boot architecture.

```text
                    ┌───────────────────────┐
                    │      Web Browser      │
                    └───────────┬───────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │ Controllers / REST API│
                    └───────────┬───────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │      Services         │
                    │ Business Logic         │
                    └───────────┬───────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
          ┌──────────────────┐    ┌──────────────────┐
          │ DSA Layer        │    │ Repository Layer │
          │ Allocation       │    │ Spring Data JPA  │
          │ Queue            │    └─────────┬────────┘
          │ Search           │              │
          │ Stack            │              ▼
          └──────────────────┘          ┌───────┐
                                        │  H2   │
                                        └───────┘
```

The DSA layer is deliberately isolated in:

```text
com.smartparking.ds
```

This reduces coupling between algorithmic logic and framework-specific code.

---

# 11. Repository Structure

High-level structure:

```text
Smart-Parking-Allocation/
│
├── ASSUMPTIONS.md
├── README.md
├── pom.xml
│
├── src/
│   ├── main/
│   │   ├── java/com/smartparking/
│   │   │   ├── config/
│   │   │   ├── controller/
│   │   │   │   ├── admin/
│   │   │   │   ├── operator/
│   │   │   │   ├── viewer/
│   │   │   │   └── auth/
│   │   │   ├── ds/
│   │   │   ├── dto/
│   │   │   ├── exception/
│   │   │   ├── model/
│   │   │   ├── repository/
│   │   │   ├── security/
│   │   │   ├── service/
│   │   │   └── util/
│   │   │
│   │   └── resources/
│   │       ├── application.properties
│   │       └── static/
│   │           ├── admin/
│   │           ├── operator/
│   │           ├── viewer/
│   │           ├── css/
│   │           ├── js/
│   │           └── images/
│   │
│   └── test/
│       ├── java/com/smartparking/
│       │   ├── ds/
│       │   ├── frontend/
│       │   ├── qa/
│       │   ├── security/
│       │   └── service/
│       └── resources/
│
└── .gitignore
```

---

# 12. Technology Stack

## Backend

- Java 21
- Spring Boot 3.3.x
- Spring MVC
- Spring Data JPA
- Spring Security
- Bean Validation

## Database

- H2

## Frontend

- HTML5
- CSS3
- Vanilla JavaScript
- ES Modules

No frontend framework is required for the current implementation.

## Build

- Apache Maven

## Testing

- JUnit 5
- Spring Boot Test
- Spring Security Test
- MockMvc

---

# 13. Running the Project Locally

## Prerequisites

Install:

```text
JDK 21+
Maven 3.8+
Modern Web Browser
```

Verify:

```bash
java -version
mvn -version
```

---

## Clone Repository

```bash
git clone https://github.com/Suhani-Parashar/Smart-Parking-Allocation.git
cd Smart-Parking-Allocation
```

---

## Build and Run Tests

```bash
mvn clean verify
```

This executes the project's automated test suite and build lifecycle.

---

## Start the Application

```bash
mvn spring-boot:run
```

Default local address:

```text
http://localhost:8080
```

---

# 14. Application Portals

### Gateway

```text
http://localhost:8080/
```

### Login

```text
http://localhost:8080/login.html
```

### Super Admin Portal

```text
http://localhost:8080/admin/index.html
```

### Operator Portal

```text
http://localhost:8080/operator/index.html
```

### Viewer Portal

```text
http://localhost:8080/viewer/index.html
```

---

# 15. Demo Environment

The `dev` and `test` profiles provide seeded demonstration data.

### Demo Users

| Username | Role |
|---|---|
| `admin` | `SUPER_ADMIN` |
| `operator` | `OPERATOR` |
| `viewer` | `VIEWER` |

Passwords are profile/configuration driven and should be treated as **demo credentials only**.

For development, the application supports seed password configuration through application properties/environment configuration.

> Do not use demo credentials in a production deployment.

---

# 16. Example Parking Layout

The baseline development dataset contains:

| Slot | Type | Zone | Floor |
|---|---|---|---:|
| `A-01` | NORMAL | A | 0 |
| `A-02` | NORMAL | A | 0 |
| `A-03` | EV | A | 0 |
| `A-04` | LARGE | A | 0 |
| `A-05` | ACCESSIBLE | A | 0 |

---

# 17. Example End-to-End Scenario

Consider the following sequence.

### Vehicle 1

```text
Type: CAR
Priority: NORMAL
```

Compatible slot:

```text
A-01
```

Result:

```text
ALLOCATED → A-01
```

### Vehicle 2

```text
Type: EV
Priority: NORMAL
```

Preferred slot:

```text
A-03
```

Result:

```text
ALLOCATED → A-03
```

### Vehicle 3

```text
Type: SUV
Priority: NORMAL
```

Compatible slot:

```text
A-04
```

Result:

```text
ALLOCATED → A-04
```

### Vehicle 4

```text
Type: SUV
Priority: NORMAL
```

No compatible LARGE slot remains.

Result:

```text
WAITING QUEUE → Position #1
```

### Vehicle 5

```text
Type: SUV
Priority: HIGH
```

Result:

```text
WAITING QUEUE → Position #1
```

The higher-priority vehicle is ordered ahead of the normal-priority vehicle.

When `A-04` is released:

```text
A-04 AVAILABLE
       │
       ▼
PriorityQueue
       │
       ▼
HIGH-priority SUV
       │
       ▼
Automatic Reallocation
       │
       ▼
A-04 → SUV
```

---

# 18. Testing Strategy

The repository contains automated tests covering multiple layers.

### DSA Tests

Tests cover:

- priority queue ordering;
- FIFO behavior;
- parking allocation;
- recent-history stack;
- slot search;
- vehicle lookup;
- queue reallocation.

### Security Tests

Tests cover:

- role isolation;
- anonymous access;
- administrative authorization;
- operator authorization;
- viewer restrictions;
- CSRF-protected operations;
- login lockout behavior.

### Privacy Tests

The repository includes tests designed to detect accidental data leakage from viewer-facing DTOs and JSON responses.

### Integration Tests

Parking service integration scenarios verify the interaction between:

```text
Controller
   ↓
Service
   ↓
Repository
   ↓
Database
```

### Hardening Tests

The project also includes tests for scenarios such as:

- duplicate submissions;
- startup waiting-queue reconstruction;
- incompatible queue entries;
- static error pages;
- validation failures;
- maintenance slots;
- automatic reallocation.

---

# 19. Engineering Decisions

## Why PriorityQueue?

A parking waiting list is not simply FIFO.

A vehicle with a higher operational priority may need to be served before a normal-priority vehicle.

Therefore:

```text
Priority + Arrival Order
```

is better represented by a priority queue than a plain FIFO queue.

---

## Why HashMap?

Vehicle lookup is frequently performed by normalized registration number.

A hash table provides average:

```text
O(1)
```

lookup.

---

## Why ArrayDeque?

Recent parking exits follow a natural LIFO pattern:

```text
Most Recent Exit
      ↓
     TOP
```

`ArrayDeque` provides efficient stack-style operations without requiring a custom low-level array implementation.

---

## Why Separate DTOs?

Returning JPA entities directly can unintentionally expose fields that a particular user role should never see.

Role-specific DTOs provide a clear boundary:

```text
Entity
  ↓
Mapping Layer
  ↓
Role-specific DTO
  ↓
API Response
```

---

# 20. Design Philosophy

The interface follows a visual system inspired by:

- road markings;
- highway signage;
- parking infrastructure;
- operational dashboards.

The project deliberately uses a framework-light frontend and keeps styling centralized through reusable design tokens.

The visual design is intended to support the domain rather than act as decoration.

---

# 21. Current Scope

The current implementation focuses on:

```text
Parking Allocation
Queue Management
Vehicle Management
Slot Management
Parking Sessions
Role-Based Security
History
Statistics
Audit Logging
Privacy-Aware DTOs
Automated Testing
```

---

# 22. Current Limitations

This repository intentionally does **not** claim capabilities that are outside its current implementation scope.

### Hardware Integration

No physical IoT parking sensors, barrier controllers, ultrasonic sensors, or loop detectors are connected.

Vehicle entry and exit are currently initiated through the application/API.

### Payments

Payment gateways and tariff calculation are outside the current scope.

### ANPR / Computer Vision

Automatic number plate recognition using CCTV or computer vision is not currently implemented.

Vehicle registration numbers are entered and normalized through the application.

### Database Deployment

The current project uses H2 for development and evaluation.

A production deployment should consider a managed relational database such as PostgreSQL.

### Enterprise Identity

The application currently uses application-level authentication rather than enterprise SSO providers such as SAML/OAuth2/OIDC.

---

# 23. Future Engineering Roadmap

Potential production extensions include:

```text
PostgreSQL
    ↓
Redis / distributed queue state
    ↓
Dockerized deployment
    ↓
Cloud deployment
    ↓
IoT sensor integration
    ↓
ANPR / Computer Vision
    ↓
Dynamic occupancy telemetry
    ↓
Reservation system
    ↓
Pricing / billing
    ↓
Payment gateway
    ↓
Observability + centralized logging
    ↓
CI/CD
```

These are roadmap items, not claims about the current implementation.

---

# 24. What This Project Demonstrates

From an engineering and hiring perspective, this project demonstrates practical exposure to:

### Backend Engineering

- Java 21
- Spring Boot
- REST APIs
- service-layer architecture
- repository pattern
- JPA persistence
- validation
- exception handling

### Data Structures & Algorithms

- HashMap
- PriorityQueue
- Binary Heap concepts
- Stack
- Queue
- Linear Search
- Comparator-based ranking
- complexity analysis

### Security Engineering

- Spring Security
- RBAC
- session authentication
- CSRF protection
- BCrypt
- access-control testing
- data exposure controls
- brute-force protection

### Software Quality

- unit testing
- integration testing
- MockMvc
- security testing
- regression-oriented tests
- hardening scenarios
- explicit assumptions and limitations

### Frontend Engineering

- semantic HTML
- CSS architecture
- JavaScript modules
- role-specific application portals
- API integration
- responsive operational interfaces

---

# 25. Engineering Principles Used

The project follows several engineering principles:

```text
Separation of Concerns
        +
Least Privilege
        +
Deterministic Algorithms
        +
Server-Side Validation
        +
Explicit Data Boundaries
        +
Automated Testing
        +
Graceful Failure
        +
Documented Assumptions
```

The goal is to make important system behavior understandable from both the implementation and the documentation.

---

# 26. Suggested Repository Walkthrough for Reviewers

A reviewer interested in the algorithmic core can start with:

```text
src/main/java/com/smartparking/ds/
```

Recommended reading order:

```text
ParkingAllocationManager.java
        ↓
SlotRankComparator.java
        ↓
WaitingVehiclePriorityQueue.java
        ↓
VehicleLookupManager.java
        ↓
ParkingHistoryStack.java
        ↓
SlotSearchManager.java
```

For security:

```text
config/SecurityConfig.java
        ↓
security/
        ↓
test/security/
```

For business workflows:

```text
service/
        ↓
controller/
        ↓
repository/
        ↓
model/
```

For testing:

```text
src/test/java/com/smartparking/
```

---

# 27. Academic Context

This project was developed as a **Data Structures and Algorithms-oriented application project**.

Instead of demonstrating algorithms in isolation, the project maps each DSA concept to a real application requirement.

For example:

```text
HashMap
   ↓
Fast Vehicle Lookup

PriorityQueue
   ↓
Priority-based Waiting

Stack
   ↓
Recent Exits

Linear Search
   ↓
Slot Search

Comparator
   ↓
Best Compatible Slot Ranking
```

This creates a direct relationship between theoretical DSA concepts and software engineering use cases.

---

# 28. Project Status

**Status:** Active academic engineering project / portfolio project

**Primary objective:** Demonstrate the integration of data structures, algorithms, backend engineering, security, persistence, and testing within a realistic business workflow.

---

# 29. Repository

**GitHub Repository**

https://github.com/Suhani-Parashar/Smart-Parking-Allocation

---

# 30. Author

**Suhani Parashar**

Computer Science Engineering Student

Primary areas demonstrated in this project:

```text
Java
Spring Boot
Backend Development
Data Structures & Algorithms
Database Development
Security
Software Testing
Frontend Engineering
```

---

# 31. Final Note for Reviewers

The most important part of this project is not the parking UI.

It is the decision-making engine behind it.

The application converts a real-world operational problem into explicit computational rules:

```text
Vehicle
  ↓
Validation
  ↓
Compatibility
  ↓
Allocation
  ↓
Priority Queue
  ↓
Automatic Reallocation
  ↓
Persistence
  ↓
Audit / History
```

Every major decision is represented in code, separated into testable components, and supported by automated verification.

That makes the project suitable for evaluating not only frontend implementation, but also understanding of:

**algorithms → backend architecture → security → data modeling → testing → engineering trade-offs.**
