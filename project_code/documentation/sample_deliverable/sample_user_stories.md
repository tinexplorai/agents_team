# User Stories

> **Created by:** Agent_01_PO  
> **Phase:** 1  
> **Input:** project_setup/step_1_project/step_1_project.md, project_setup/step_2_requirements/

---

## US-1: User Registration
**As a** new user, **I want to** create an account with email and password, **so that** I can access the platform.

### Acceptance Criteria
- [ ] AC1: User can enter email, password (min 8 chars), and confirm password
- [ ] AC2: System validates email format and password strength
- [ ] AC3: System sends verification email after successful registration
- [ ] AC4: User receives error message if email already exists
- [ ] AC5: User is redirected to login page after email verification

---

## US-2: User Login
**As a** registered user, **I want to** log in with my credentials, **so that** I can access my account.

### Acceptance Criteria
- [ ] AC1: User can enter email and password
- [ ] AC2: System validates credentials and creates session
- [ ] AC3: User is redirected to dashboard on successful login
- [ ] AC4: User sees error message for invalid credentials
- [ ] AC5: User can request password reset if forgotten

---

## US-3: View Dashboard
**As a** logged-in user, **I want to** see my dashboard, **so that** I can access main features.

### Acceptance Criteria
- [ ] AC1: Dashboard displays user profile summary
- [ ] AC2: Dashboard shows recent activity (last 10 items)
- [ ] AC3: Dashboard provides quick access to main features
- [ ] AC4: Dashboard loads within 2 seconds

---

## Assumptions

- Email verification is required before first login
- Session expires after 24 hours of inactivity
- Password reset link expires after 1 hour
