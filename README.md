# NEXORA — Team Resource Booking Platform

> **Reserve space. Coordinate better. Work smarter.**

NEXORA is a production-style internal resource booking platform designed for teams to discover, reserve, and manage shared resources such as meeting rooms, equipment, workspaces, and collaboration areas.

The project focuses on reliable scheduling, database-level conflict prevention, role-based access control, recurring bookings, approval workflows, and real-time notifications.

---

# Overview

NEXORA allows team members to:

* Discover available resources
* View daily and weekly availability
* Create one-time or recurring bookings
* Track upcoming and previous bookings
* Receive real-time booking notifications
* Cancel their bookings

Administrators can additionally:

* Create and manage resources
* Configure approval requirements
* Approve or reject booking requests
* Monitor resource utilization
* View booking analytics

The application is designed around database-enforced correctness rather than relying only on client-side validation.

---

# Key Features

### Authentication & Authorization

* Supabase Authentication
* Admin and regular member roles
* Row Level Security (RLS)
* Server-side authorization checks

### Resource Management

* Create and manage resources
* Resource categories
* Capacity and location
* Amenities
* Resource images
* Approval-required configuration

### Availability

* Day and week calendar views
* Resource filtering
* Visual booking states
* Available and unavailable time slots

### Booking

* One-time bookings
* Conflict prevention
* Booking status tracking
* Booking cancellation
* Booking history

### Recurring Bookings

* Weekly recurrence
* Configurable number of occurrences
* Individual booking records for each occurrence
* Recurrence group tracking

### Approval Workflow

* Pending bookings for approval-required resources
* Admin approval
* Admin rejection
* Booking status state machine

### Real-Time Notifications

* Booking confirmations
* Rejections
* Conflict notifications
* Approval notifications
* Supabase Realtime subscriptions

### Calendar Export

* `.ics` export for confirmed bookings
* Local calendar file generation
* No external calendar API integration

### Admin Dashboard

* Booking statistics
* Resource utilization
* Pending approvals
* Booking trends

---

# Tech Stack

| Layer           | Technology                    |
| --------------- | ----------------------------- |
| Framework       | Next.js 16.2                  |
| Language        | TypeScript 6.0.3              |
| Styling         | Tailwind CSS v4               |
| Validation      | Zod 4.4.3                     |
| Authentication  | Supabase Auth                 |
| Database        | PostgreSQL 17 / Supabase      |
| Authorization   | PostgreSQL Row Level Security |
| Real-Time       | Supabase Realtime             |
| Deployment      | Vercel                        |
| Calendar Export | `ics`                         |

---

# Architecture

```text
                    ┌──────────────────────┐
                    │      NEXORA UI       │
                    │  Next.js App Router  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Server/API Boundary  │
                    │ Zod Validation       │
                    │ Authorization        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       Supabase       │
                    │                      │
                    │ PostgreSQL            │
                    │ Supabase Auth        │
                    │ RLS                  │
                    │ Realtime             │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
          Resources         Bookings       Notifications
              │                │                │
              └────────────────┴────────────────┘
                               │
                               ▼
                    Database Constraints
                    & Transaction Logic
```

---

# Database Design

The main entities are:

### Profiles

Stores authenticated user information and role.

```text
profiles
- id
- full_name
- email
- avatar_url
- role
- created_at
```

Roles:

```text
admin
member
```

### Resources

Represents bookable resources.

```text
resources
- id
- name
- description
- capacity
- category
- location
- image_url
- amenities
- requires_approval
- is_active
- created_at
- updated_at
```

### Bookings

Represents individual reservations.

```text
bookings
- id
- resource_id
- user_id
- title
- description
- start_time
- end_time
- time_range
- status
- recurrence_group_id
- created_at
- updated_at
```

Booking states:

```text
pending
approved
rejected
cancelled
```

### Notifications

Stores user-specific application notifications.

```text
notifications
- id
- user_id
- booking_id
- type
- title
- message
- is_read
- created_at
```

---

# How Double Booking Is Prevented

## Database-Level Conflict Prevention

Preventing overlapping bookings is one of the most important parts of NEXORA.

The application does **not** rely only on a frontend availability check.

A simple approach such as:

```text
1. Check whether the slot is available
2. If available, insert booking
```

is unsafe because two requests can perform the check at almost the same time.

For example:

```text
User A → Check → Available
User B → Check → Available

User A → Insert
User B → Insert
```

This can result in a double booking.

NEXORA instead enforces the rule directly in PostgreSQL.

---

## PostgreSQL Range Type

Each booking has a timezone-aware time range using PostgreSQL `tstzrange`.

Conceptually:

```text
[start_time, end_time)
```

The range represents the complete occupied period of the resource.

---

## EXCLUDE Constraint

PostgreSQL's exclusion constraint is used together with the `btree_gist` extension.

The constraint enforces:

```text
Same resource
+
Overlapping time ranges
=
NOT ALLOWED
```

Conceptually:

```text
EXCLUDE USING gist (
    resource_id WITH =,
    time_range WITH &&
)
```

The `=` operator checks whether the resource is the same.

The `&&` operator checks whether two time ranges overlap.

Therefore, two bookings for the same resource cannot overlap.

---

## Concurrent Booking Scenario

Suppose:

```text
Resource: Aurora Conference Room

User A:
10:00 → 11:00

User B:
10:00 → 11:00
```

Both requests arrive almost simultaneously.

The database becomes the final authority.

One transaction successfully creates the booking.

The second insert violates the exclusion constraint and fails.

The application catches the database conflict and returns a user-friendly response:

> "That slot was just booked. Please choose another time."

This means the system remains correct even when multiple users submit requests at nearly the same time.

---

# Recurring Booking Design

NEXORA supports recurring bookings such as:

```text
Every Tuesday
3:00 PM – 4:00 PM
For 8 weeks
```

Recurring bookings are **materialized into individual booking rows**.

For example, an 8-week recurrence creates:

```text
Booking 1 → Tuesday Week 1
Booking 2 → Tuesday Week 2
Booking 3 → Tuesday Week 3
...
Booking 8 → Tuesday Week 8
```

Each occurrence is an independent booking record.

The individual records share a common:

```text
recurrence_group_id
```

This allows the system to identify which bookings belong to the same recurring series while still allowing each occurrence to participate independently in conflict detection and status management.

This approach also means every generated occurrence is subject to the same database-level overlap protection as a normal booking.

---

# Booking State Machine

Booking status is represented explicitly.

```text
                 ┌───────────┐
                 │  PENDING  │
                 └─────┬─────┘
                       │
              ┌────────┴────────┐
              ▼                 ▼
          APPROVED            REJECTED
              │
              ▼
          CANCELLED
```

Possible states:

* `pending`
* `approved`
* `rejected`
* `cancelled`

A booking requiring administrator approval starts as `pending`.

An administrator can approve or reject it.

A confirmed booking can subsequently be cancelled.

---

# Approval Workflow

For resources where:

```text
requires_approval = true
```

the workflow is:

```text
Member creates booking
        ↓
Booking = pending
        ↓
Admin reviews request
        ↓
    ┌───┴────┐
    ▼        ▼
Approve    Reject
    │        │
    ▼        ▼
Approved   Rejected
```

Rejected bookings do not remain as active reservations.

---

# Row Level Security

NEXORA uses PostgreSQL Row Level Security to protect application data.

Authorization is not implemented only through frontend UI controls.

For example:

* Members can access their permitted booking data.
* Admin-only operations are protected at the database/API level.
* A regular member cannot approve a booking simply by manually calling the API.
* Resource and booking access is controlled through database policies.

This provides a second layer of protection beyond frontend route restrictions.

---

# Real-Time Notifications

NEXORA uses Supabase Realtime rather than periodic polling.

When a relevant database event occurs, the application receives the update through a realtime subscription.

Examples:

```text
Booking approved
Booking rejected
Booking conflict
Approval requested
Booking cancelled
```

The notification center updates without requiring the user to refresh the page.

---

# Validation

Zod is used at API/server boundaries to validate untrusted input.

Examples:

* Booking dates
* Start/end times
* Resource IDs
* Resource creation
* Recurrence configuration
* Approval/rejection requests

Client-side validation improves user experience, but server-side validation remains the authoritative validation layer.

---

# Security

Security considerations include:

* Supabase Authentication
* PostgreSQL RLS
* Server-side authorization
* Zod input validation
* Database-level conflict prevention
* Environment variables for secrets
* No sensitive credentials committed to Git
* Explicit booking state transitions

---

# UI / UX

NEXORA uses a modern SaaS interface designed around:

* Dark/light themes
* Responsive layouts
* Interactive calendar
* Resource discovery
* Real-time notifications
* Animated interactions
* Loading skeletons
* Empty states
* Error states
* Accessible forms
* Keyboard-friendly interactions

The visual system uses a dark foundation with violet, indigo, cyan, and neutral accents to distinguish important states without relying only on color.

---

# Responsive Design

The interface is optimized for:

* Desktop
* Laptop
* Tablet
* Mobile

The navigation, calendar, booking workflow, resource cards, filters, and dashboards adapt to smaller screens.

---

# Testing

The project should be tested for:

### Booking

* Valid booking
* Invalid time ranges
* Cancellation
* Approval-required resources

### Conflict Detection

* Exact overlapping bookings
* Partial overlaps
* Nested overlaps
* Different resources at the same time
* Adjacent non-overlapping bookings

Example:

```text
10:00–11:00
11:00–12:00
```

These two bookings should be allowed because they do not overlap.

### Concurrency

Multiple simultaneous booking requests are tested against the same resource and time range.

The expected result is:

```text
One request → succeeds
Other request → database conflict
```

---

# Local Development

## Prerequisites

* Node.js 24+
* npm
* Supabase project
* Git

## Installation

```bash
git clone YOUR_REPOSITORY_URL
cd nexora

npm install
```

Create `.env.local`:

```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
```

Run the development server:

```bash
npm run dev
```

Open:

```text
http://localhost:3000
```

---

# Environment Variables

Never commit `.env.local`.

Required environment variables:

```text
NEXT_PUBLIC_SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY
```

Add any additional server-only Supabase variables required by the implementation through the deployment platform.

---

# Deployment

The application is deployed using Vercel.

Deployment process:

```text
GitHub
   ↓
Vercel
   ↓
Next.js production build
   ↓
Supabase PostgreSQL
```

Environment variables are configured through Vercel rather than committed to the repository.

---

# Out of Scope

The following features are intentionally not implemented:

* Google Calendar synchronization
* Outlook synchronization
* External calendar OAuth
* Email reminders
* SMS reminders
* Payment processing
* Multi-organization billing
* External calendar APIs

These are intentionally excluded to keep the implementation focused on the assignment requirements.

---

# Future Improvements

With another week of development, I would focus on:

* Advanced resource utilization insights
* Bulk resource management
* More sophisticated recurring booking management
* Calendar drag-and-drop rescheduling
* Audit logs for administrative actions
* More comprehensive automated integration tests
* Improved accessibility auditing
* Performance monitoring and observability
* Fine-grained notification preferences

---

# Project Status

**Status:** Completed Prototype

Built as a candidate screening project demonstrating:

* Full-stack development
* PostgreSQL database design
* Supabase authentication
* Row Level Security
* Database-level concurrency control
* Real-time application functionality
* Scheduling logic
* Responsive UI/UX
* Production deployment

---

## Author

**SHALU KUMAWAT**

B.Tech Computer Science & Engineering



LinkedIn: `[YOUR LINKEDIN PROFILE]`
