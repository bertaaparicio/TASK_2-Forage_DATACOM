# Kudos System Specification

## Overview
Build a “Kudos” feature for the internal employee portal:
- Employees can send kudos (short appreciation messages) to colleagues.
- A public dashboard feed shows recent kudos.
- Admins can moderate inappropriate content (hide or delete).

---

## Functional Requirements

### Roles
- **User**: authenticated employee who can send and view kudos.
- **Admin**: authenticated user with moderation privileges.

### User Stories
1. **Send Kudos**
   - As a user, I can select a colleague from a list and write a short message of appreciation to send kudos.
2. **View Feed**
   - As a user, I can view a public feed on the main dashboard that shows recently submitted kudos.
3. **Input Validation**
   - As a user, I receive clear validation errors if I exceed the message length, omit required fields, or attempt to submit empty content.
4. **Moderate Content (NEW)**
   - As an admin, I can **hide** inappropriate kudos from the public feed.
   - As an admin, I can **delete** inappropriate kudos messages permanently.

### Acceptance Criteria

#### 1) Send Kudos
- User must be logged in.
- User can select a recipient from a list of users (excluding self).
- Message is required, trimmed, and must be **1–500 characters**.
- On success:
  - Kudos is stored in DB.
  - Kudos appears in the public feed (if visible).
  - User sees confirmation.

#### 2) View Feed
- Dashboard shows recent kudos sorted by newest first.
- Feed shows: sender name, recipient name, message, timestamp.
- Feed shows only kudos where `is_visible = true`.
- Pagination or “load more” supported (e.g., 20 at a time).

#### 3) Validation & Safety
- Reject empty/whitespace-only messages.
- Basic server-side sanitization/escaping to prevent XSS in rendered messages.
- Rate limiting (lightweight): e.g., max 10 kudos per user per hour (optional but recommended).

#### 4) Admin Moderation (NEW)
- Admin can hide a kudos:
  - Hidden kudos no longer appears in the public feed.
  - Hidden kudos remains in DB for audit/traceability.
- Admin can delete a kudos:
  - Deleted kudos is removed from DB (or soft-delete if preferred).
- Admin actions are logged (who, when, what action).

---

## Technical Design

### Assumptions / Constraints
- Existing app already has user authentication.
- We have a Users table (or identity provider) with unique user IDs.
- Internal web app stack can be implemented as:
  - Backend API (e.g., Python Flask/FastAPI or Node)
  - Frontend (e.g., React) or server-rendered templates
  - SQL database (e.g., PostgreSQL/SQLite for demo)

### Data Model

#### Table: `kudos`
| Field | Type | Notes |
|------|------|------|
| id | UUID / INTEGER | Primary key |
| sender_user_id | VARCHAR / UUID | FK to users |
| recipient_user_id | VARCHAR / UUID | FK to users |
| message | TEXT | 1–500 chars (enforced in API/DB constraints where possible) |
| created_at | TIMESTAMP | default now |
| is_visible | BOOLEAN | **default true** (NEW requirement) |
| moderated_by | VARCHAR / UUID | nullable FK to users (admin) |
| moderated_at | TIMESTAMP | nullable |
| moderation_reason | TEXT | nullable (optional) |

**Indexes**
- `created_at` for feed sorting
- `(is_visible, created_at)` composite index for feed filtering
- `recipient_user_id` index (optional, for future “my kudos” views)

### API Design

#### Auth
All endpoints require authentication. Admin endpoints require admin role.

#### Endpoints

1) **Get user list (for recipient selection)**
- `GET /api/users`
- Response: list of `{ user_id, display_name }`

2) **Create kudos**
- `POST /api/kudos`
- Body:
  ```json
  { "recipient_user_id": "U123", "message": "Great job on the release!" }
